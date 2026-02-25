# Skills

Shared Claude skills for the appgarden team.

## Installing a skill

Claude discovers skills automatically from `~/.claude/skills/`. To install a skill from this repo, clone it and symlink the skill you want:

```bash
# Clone this repo into your appgarden directory (if you haven't already)
git clone https://github.com/appgarden-io/skills.git

# Symlink the skill into your local Claude skills directory
ln -s "$(pwd)/skills/start-new-project" ~/.claude/skills/start-new-project
```

The symlink approach keeps everything in sync — when the repo is updated with `git pull`, your local skill updates too.

Alternatively, you can copy a skill directly:

```bash
cp -r skills/start-new-project ~/.claude/skills/
```

## Available skills

| Skill | Description |
|-------|-------------|
| [start-new-project](./start-new-project) | Scaffolds a new prototype project from the appgarden template |

## Adding a new skill

1. Create a new directory at the root of this repo with your skill name in kebab-case (e.g. `my-new-skill/`)
2. Add a `SKILL.md` file inside that directory. This is the only required file. It needs YAML frontmatter with `name` and `description`, followed by the skill instructions in markdown:

   ```markdown
   ---
   name: my-new-skill
   description: What the skill does and when to use it.
   ---

   # My New Skill

   Instructions for Claude go here.
   ```

3. If your skill needs supporting files (scripts, reference docs, templates), add them in subdirectories:

   ```
   my-new-skill/
   ├── SKILL.md          # Required — instructions and metadata
   ├── scripts/          # Optional — executable helpers
   ├── references/       # Optional — docs loaded into context as needed
   └── assets/           # Optional — templates, icons, etc.
   ```

4. Update the **Available skills** table in this README
5. Commit and push to `main`

### Tips for writing good skills

- **Keep `SKILL.md` under 500 lines.** If it's getting long, move detail into `references/` files and point to them from the main doc.
- **Write the description for triggering.** The `description` field is how Claude decides whether to use the skill, so include the kinds of phrases or contexts that should activate it.
- **Explain the why, not just the what.** Claude follows instructions better when it understands the reasoning behind them.
- **Use examples.** Show what good input/output looks like — concrete examples beat abstract rules.
