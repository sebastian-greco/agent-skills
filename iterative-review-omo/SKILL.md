---
name: iterative-review-omo
description: Iterative code review loop for oh-my-opencode — spawns a fresh Oracle reviewer, triages feedback by severity, fixes issues with individual commits, and loops until clean. Pauses every 5 rounds for confirmation.
---

# Iterative Review Loop (oh-my-opencode)

> **Requires:** [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) with `oracle` and `hephaestus` agents configured.

Spawn a fresh Oracle reviewer, triage feedback by severity, fix relevant issues with individual commits, and loop until the code is clean. Pause every 5 rounds to confirm continuation.

## Trigger

Use this skill when the user says any of:
- "review my code", "iterative review", "review loop"
- "spawn oracle to review", "review until clean"
- `/iterative-review`

## Arguments

`[context]` (optional) — brief description of what was just built. If omitted, context is inferred automatically.

## Step 0 — Gather Context

Determine the feature description using this priority order:

**1. Explicit argument** — if the user passed context, use it verbatim. Skip to Step 1.

**2. Session context** — if you have been actively working on this feature in the current conversation (you implemented, modified, or discussed specific files/logic in this session), you already have the context. Summarize what was built in 2-3 sentences from memory. Skip the git commands. Skip to Step 1.

**3. Git fallback** — only if neither of the above applies (cold start, someone else's code, long unrelated session):

```bash
git diff HEAD~1..HEAD
git log --oneline -5
```

Synthesize a 2-3 sentence feature description from the diff and log.

## Step 1 — Spawn Oracle Reviewer

Spawn a fresh `oracle` agent via `task()` (`run_in_background=false`) with this framing:

> You are a senior code reviewer. Review the following diff and provide structured feedback.
>
> **What was built:** {feature_description}
>
> **Diff:**
> ```
> {git_diff}
> ```
>
> Return feedback in this exact format:
>
> ### Critical (must fix before merge)
> - [issue]: [specific location] — [why it matters]
>
> ### Important (should fix)
> - [issue]: [specific location] — [why it matters]
>
> ### Minor (nice to have)
> - [issue]: [specific location] — [suggestion]
>
> ### Praise (what's good)
> - [what works well]
>
> If you have zero Critical and zero Important issues, end your response with exactly: `LGTM ✅`

## Step 2 — Triage Feedback

**Critical / Important → Fix it (unless you have a strong technical reason not to):**
- Agree: fix immediately → commit `fix(review): <short description> [round N]`
- Disagree: state technical reasoning in 1-2 sentences, do NOT fix → log as `Skipped: {reason}`
- Unclear: ask the user before proceeding

**Minor → Fix if trivial (< 5 lines), log otherwise:**
- Fixed: commit `fix(review): <short description> [round N] [minor]`
- Skipped: log as `Deferred minor: {description}`

**Irrelevant / Wrong:**
- Log as `Skipped (not applicable): {reason}`

After each round, print a triage summary:

```
Round N summary:
  ✅ Fixed (Critical):    X issues
  ✅ Fixed (Important):   Y issues
  ✅ Fixed (Minor):       Z issues
  ⏭️  Skipped (rejected):  A issues — [brief reasons]
  ⏭️  Deferred (minor):    B issues
```

## Step 3 — Check Exit Condition

**Exit** if Oracle's response contains `LGTM ✅` OR zero Critical/Important issues remain after triage.

```
✅ Iterative review complete.
   Rounds:                N
   Total fixes committed: M
   Deferred minors:       K (logged above)
```

Stop. Do not spawn another round.

## Step 4 — Loop or Checkpoint

**Round < 5 and not clean:** go back to Step 1 with a fresh Oracle spawn.

**Round is a multiple of 5 and not clean:** pause and ask:

```
⏸️  5 rounds completed (N total).
   Issues fixed this batch: X
   Oracle still has Critical/Important comments.

   Continue another 5 rounds? (y/n)
```

- Yes: continue from Step 1, tracking cumulative totals
- No: stop, show full session summary including all deferred and skipped items

## Commit Convention

```
fix(review): <issue description> [round N]
fix(review): <issue description> [round N] [minor]
```

Examples:
- `fix(review): add null check for user input [round 1]`
- `fix(review): extract magic number to constant [round 2] [minor]`

## Agent Assignment (oh-my-opencode specific)

- **Code changes** (refactoring, bug fixes, new logic): delegate to `hephaestus` via `task()`
- **Architecture / logic decisions**: handle yourself or consult `oracle`
- **Simple fixes** (typos, imports, formatting): apply directly with edit tools

## Anti-Patterns

**NEVER:**
- Bundle multiple issues into one commit — one commit per issue
- Accept Oracle feedback blindly — verify against the actual codebase first
- Spawn Oracle without the real diff — it cannot review what it cannot see
- Continue past a 5-round checkpoint without explicit user confirmation

**NEVER stop early because:**
- "The code looks fine to me" — if Oracle has Critical/Important items, address or explicitly reject with reasoning
- "Only minor issues left" — minors are fine to defer; Critical/Important are not
