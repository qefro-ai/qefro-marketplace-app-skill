# Qefro Marketplace App skill

Skill for Cursor and Claude that generates metadata-only Qefro Marketplace Apps. The agent writes YAML (manifest, entities, workflows, staff UI). The Qefro Runtime executes it. Generated apps belong in [`qefro-marketplace-apps`](https://github.com/qefro-ai/qefro-marketplace-apps), not in this repo.

## Layout

```
skills/qefro-marketplace-app/     canonical skill
.cursor/skills/qefro-marketplace-app
.claude/skills/qefro-marketplace-app
.agents/skills/qefro-marketplace-app
```

The three dotted directories are symlinks to the canonical skill, so Cursor and Claude load the same instructions.

## Use it

Open this repo, or the Qefro workspace that links the skill, and ask for an app:

- "Generate a salon booking Marketplace App"
- "Add cancellation to clinic-pro"
- "Audit field-service against the metadata contract"

The agent should read `skills/qefro-marketplace-app/SKILL.md`, follow Restaurant Pro or Real Estate Pro, write YAML under `qefro-marketplace-apps/apps/<app-id>/`, and run `python3 scripts/validate_apps.py`.

## Install beside an existing checkout

From the parent of this repo (the `qefro` folder):

```bash
ln -sfn ../../qefro-marketplace-app-skill/skills/qefro-marketplace-app .cursor/skills/qefro-marketplace-app
ln -sfn ../../qefro-marketplace-app-skill/skills/qefro-marketplace-app .claude/skills/qefro-marketplace-app
ln -sfn ../../qefro-marketplace-app-skill/skills/qefro-marketplace-app .agents/skills/qefro-marketplace-app
```

User-level install, so the skill is available outside that workspace:

```bash
ln -sfn /path/to/qefro-marketplace-app-skill/skills/qefro-marketplace-app ~/.claude/skills/qefro-marketplace-app
```

Cursor personal skills use the user Agent Store `skills/` directory the same way.
