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

## How it works

Each skill is a folder with a `SKILL.md` file. The `skills` CLI injects the skill content into your agent's context at the start of a session.

```
agent-skills/
├── iterative-review/
│   └── SKILL.md
└── iterative-review-omo/
    └── SKILL.md
```
