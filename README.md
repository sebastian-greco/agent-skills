# agent-skills

A collection of reusable skills for AI coding agents, installable via [skills.sh](https://skills.sh).

## Skills

### `iterative-review`

Iterative code review loop — spawns a fresh reviewer subagent, triages feedback by severity, fixes issues with individual commits, and loops until clean. Works with **any coding agent** (Claude Code, Codex, Cursor, Copilot, etc.).

```bash
npx skills add sebs/agent-skills --skill iterative-review
```

### `iterative-review-omo`

Same loop, optimized for **[oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode)** — hardcodes `oracle` as the reviewer and delegates code changes to `hephaestus` via `task()`.

```bash
npx skills add sebs/agent-skills --skill iterative-review-omo
```

---

### `address-pr-comments`

Fetch all open PR review comments, triage by criticality, fix every issue (one commit per fix), push, then reply in-thread to each comment thread with the work done. Works with **any coding agent**.

```bash
npx skills add sebs/agent-skills --skill address-pr-comments
```

### `address-pr-comments-omo`

Same workflow, optimized for **oh-my-opencode** — delegates fixes to `hephaestus` and uses the correct GitHub API endpoint for in-thread replies.

```bash
npx skills add sebs/agent-skills --skill address-pr-comments-omo
```

---

### `work-on-issue`

Start working on a GitHub issue — create a linked branch, plan implementation in phases, execute one commit per phase, and open a draft PR at the end. Never merges without explicit user approval. Works with **any coding agent**.

```bash
npx skills add sebs/agent-skills --skill work-on-issue
```

### `work-on-issue-omo`

Same workflow, optimized for **oh-my-opencode** — `oracle` produces the phased plan (mandatory, synchronous), `hephaestus` implements each phase. Ultrawork mode on by default.

```bash
npx skills add sebs/agent-skills --skill work-on-issue-omo
```

---

## Agent Distribution

| Skill | Claude Code | Codex | OpenCode (omo) |
|---|---|---|---|
| `iterative-review` | ✅ | ✅ | — |
| `iterative-review-omo` | — | — | ✅ |
| `address-pr-comments` | ✅ | ✅ | — |
| `address-pr-comments-omo` | — | — | ✅ |
| `work-on-issue` | ✅ | ✅ | — |
| `work-on-issue-omo` | — | — | ✅ |

## How it works

Each skill is a folder with a `SKILL.md` file. The `skills` CLI injects the skill content into your agent's context at the start of a session.

```
agent-skills/
├── iterative-review/
│   └── SKILL.md
├── iterative-review-omo/
│   └── SKILL.md
├── address-pr-comments/
│   └── SKILL.md
├── address-pr-comments-omo/
│   └── SKILL.md
├── work-on-issue/
│   └── SKILL.md
└── work-on-issue-omo/
    └── SKILL.md
```
