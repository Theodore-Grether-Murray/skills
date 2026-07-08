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
| [`news-brief`](skills/news-brief) | Briefs you on what's happening in the world — or any specific topic — by scraping live news via BrowserClaw in parallel tabs. Picks reputable sources that fit the topic (a general-news default trio, or beat outlets for tech / markets / a region), cross-checks them, and synthesizes a themed brief with source links and a tailored "why it matters to you." Delivers the brief in chat plus an auto-linked Google Doc opened as a tab. Ships source selectors + a proven Google-Doc-building recipe. |
| [`company-dossier`](skills/company-dossier) | Researches a company from a name or URL into a concise one-pager — what they do, product surface, funding & traction, team (via LinkedIn), and recent news — by scraping the company's site, LinkedIn, and a funding/news search via BrowserClaw in parallel tabs, cross-checking, and flagging conflicting figures. Tailors a "why it matters" to your angle (competitive intel, investor prep, hiring diligence, or lead prep). Delivers the dossier in chat plus an auto-linked Google Doc. Ships extraction payloads + the hardened Google-Doc recipe. |

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
