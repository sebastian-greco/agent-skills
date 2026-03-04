---
name: address-pr-comments
description: Check PR review comments, triage by criticality, fix every issue with individual commits, push, then reply to each comment thread with the work done. Works with any coding agent.
---

# Address PR Review Comments

Fetch all open PR review comments, triage them by criticality, fix every issue (one commit per fix), push, then reply in-thread to each comment with what was done. Fix everything — including low priority items.

## Trigger

Use this skill when the user says any of:
- "address PR comments", "fix review comments", "handle PR feedback"
- "respond to PR review", "address the review"

## Arguments

`[pr_number]` (optional) — PR number to address. If omitted, uses the PR associated with the current branch.

## Step 0 — Identify the PR

If `pr_number` was provided, use it directly. Otherwise:

```bash
# Get the PR for the current branch
gh pr view --json number,title,headRefName
```

Also fetch repo context needed for API calls:

```bash
gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"'
```

Store: `{owner}`, `{repo}`, `{pr_number}`.

## Step 1 — Fetch All Review Comments

```bash
gh api /repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --jq '.[] | select(.in_reply_to_id == null) | {id: .id, path: .path, line: .line, body: .body, diff_hunk: .diff_hunk}'
```

> **Why `select(.in_reply_to_id == null)`?** This filters to root comments only (the original thread starters). You always reply to the root, not to nested replies. Threads started by others at the same location will still get a reply.

Also fetch the PR description and diff for context:

```bash
gh pr view {pr_number} --json title,body,commits
git diff origin/main...HEAD
```

## Step 2 — Triage by Criticality

Group each root comment into one of:

| Priority | Criteria |
|---|---|
| **Critical** | Security issue, data loss risk, broken functionality, API contract violation |
| **High** | Logic bug, missing error handling, performance issue, type safety violation |
| **Medium** | Code clarity, naming, missing tests, refactoring suggestion |
| **Low** | Style, formatting, minor nitpicks, optional improvements |

Print a triage table before starting any fixes:

```
PR #N — Review Comment Triage
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Critical:  X comments
  High:      Y comments
  Medium:    Z comments
  Low:       W comments
  Total:     T comments (all will be fixed)
```

## Step 3 — Fix Every Comment

Process in priority order: Critical → High → Medium → Low.

For **each** comment:

1. Read the comment body and `diff_hunk` to understand the exact location and concern
2. Understand the full context of the change (read surrounding code if needed)
3. Implement the fix
4. Commit immediately:
   ```bash
   git add <affected files>
   git commit -m "fix(pr-{pr_number}): {short description} [{priority}]"
   ```

**One commit per comment. No batching.**

**If a comment is unclear:** make a reasonable interpretation, note it in the reply.

**If a fix is genuinely impossible** (e.g. comment refers to removed code, or contradicts another requirement): skip the fix, but still reply explaining why.

### Commit format examples:
```
fix(pr-42): add null check before accessing user.profile [critical]
fix(pr-42): rename variable for clarity [low]
fix(pr-42): extract magic number to named constant [medium]
```

## Step 4 — Push

```bash
git push
```

## Step 5 — Reply to Each Comment Thread

For every root comment addressed in Step 3, post a reply **in the same thread** using the `/replies` endpoint:

```bash
gh api \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  /repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies \
  -f body="{reply_body}"
```

**Reply format:**

```
Fixed in {commit_sha} — {1-2 sentence description of what was changed and why}.
```

For skipped comments:
```
Not fixed — {clear explanation of why this cannot or should not be changed}.
```

> **CRITICAL:** Use the `/replies` endpoint with the root comment's `id`. Do NOT use `gh pr comment` or `gh issue comment` — those create standalone PR timeline comments, not threaded replies.

## Step 6 — Summary

Print a completion summary:

```
✅ PR #N review comments addressed.
   Fixed:    X commits pushed
   Skipped:  Y (see replies for reasons)

Commits:
  abc1234 fix(pr-N): ...
  def5678 fix(pr-N): ...
```

## Anti-Patterns

**NEVER:**
- Batch multiple fixes into one commit — one commit per comment
- Skip low-priority comments — fix everything unless genuinely impossible
- Use `gh pr comment` or `gh issue comment` to reply — these add standalone comments, not threaded replies
- Reply before pushing — always push first so the commit SHA is valid in the reply
- Guess the fix without reading the surrounding code context
