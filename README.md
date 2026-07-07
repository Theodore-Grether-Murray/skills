# skills

A collection of reusable, eval-driven **skills** for coding agents (Claude Code, and any agent that supports the [Agent Skills](https://code.claude.com/docs/en/skills) format).

Each skill is a self-contained folder of instructions and supporting material that teaches an agent how to do one thing well. Skills load on demand — the agent reads a skill only when its `description` matches the task at hand — so you can keep a large library without bloating context.

## Installation

Install any skill from this repo directly into your project with the [`skills`](https://www.npmjs.com/package/skills) CLI:

```bash
npx skills add Theodore-Grether-Murray/skills --skill <skill-name>
```

Or clone the repo and copy a skill into your `.claude/skills/` directory manually:

```bash
git clone https://github.com/Theodore-Grether-Murray/skills.git
cp -r skills/skills/<skill-name> ~/.claude/skills/
```

## Available skills

| Skill | What it does |
| --- | --- |
| [`browserclaw-mcp`](skills/browserclaw-mcp) | Teaches an agent to drive a real browser through the BrowserClaw MCP — the read-vs-interact split, the snapshot→act→diff loop, when to fall back to `evaluate`, and common recipes for scraping, forms, and multi-step flows. |
| [`x-intelligence-scrape`](skills/x-intelligence-scrape) | Turns X (Twitter) into an actionable briefing by scraping the live site via BrowserClaw — what's happening, where to engage, and what to tweet. Sources only from the For You feed + Explore (never Following), filters promoted ads, and enforces 12h freshness. Ships proven scrape payloads. |

## Skill anatomy

Every skill lives in `skills/<skill-name>/` and follows this shape:

```
skills/<skill-name>/
├── SKILL.md        # Required. Frontmatter (name + description) and the instructions.
├── references/     # Optional. Deeper docs the skill points to for details.
└── evals/          # Optional. Test prompts + assertions that prove the skill works.
```

`SKILL.md` starts with YAML frontmatter:

```markdown
---
name: skill-name
description: Use when <trigger> — <what it does>. Written so an agent can decide relevance from this line alone.
---

# Skill Name

Step-by-step instructions the agent follows once the skill is loaded.
```

The `description` is the most important line: it's what an agent reads to decide whether to invoke the skill. Make it specific and trigger-oriented.

## Development approach

Skills here are meant to be **eval-driven**: each ships with realistic prompts and concrete assertions, and is iterated against them until it reliably triggers and produces the intended behavior. See [CONTRIBUTING.md](CONTRIBUTING.md) for the workflow.

## License

MIT
