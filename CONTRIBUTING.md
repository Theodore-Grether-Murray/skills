# Contributing

Thanks for adding to this skills collection. Every skill should be small, focused, and proven to work before it lands.

## Add a new skill

1. **Create the folder.** One skill per directory under `skills/`:

   ```
   skills/<skill-name>/
   ├── SKILL.md          # required
   ├── references/       # optional — supporting docs
   └── evals/            # optional — but strongly encouraged
   ```

   Use a short, kebab-case `<skill-name>` that reads like what the skill does (`responsiveness-audit`, not `skill1`).

2. **Write `SKILL.md`.** Start with frontmatter, then the instructions:

   ```markdown
   ---
   name: skill-name
   description: Use when <trigger situation> — <what the skill does>.
   ---

   # Skill Name

   Concrete, ordered steps. Assume the agent has no memory of this conversation.
   ```

   Guidelines:
   - The `description` is a triggering signal, not a summary. Lead with *when* to use it. An agent decides whether to load the skill from this line alone, so be specific about the situations that should fire it.
   - Keep `SKILL.md` focused. Push long reference material into `references/` and link to it, so the main file stays lean when loaded.
   - Write imperative, checkable steps rather than prose.

3. **Add evals.** Put realistic prompts and their expected outcomes in `evals/`. A good eval states the input prompt and a concrete assertion about what a correct run produces. Iterate the skill against these until it reliably triggers and behaves.

4. **List it in the README.** Add a row to the "Available skills" table with a one-line summary.

## Style

- Prefer clarity over cleverness — these are read by agents and humans.
- No secrets, credentials, or personal data in skill files.
- One logical change per pull request.

## Testing your skill locally

Copy the skill into a project's `.claude/skills/` (or your `~/.claude/skills/`) directory and run a prompt that should trigger it. Confirm it loads on the intended trigger and does *not* fire on unrelated tasks.
