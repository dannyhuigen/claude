# claude

Personal Claude Code commands and skills.

## Layout

- `commands/` — slash commands. Each `<name>.md` becomes `/<name>` when copied to `~/.claude/commands/`.
- `skills/` — model-invoked skills. Each `<name>/SKILL.md` becomes a skill when copied to `~/.claude/skills/`.

## Install

```bash
cp commands/*.md ~/.claude/commands/
cp -r skills/* ~/.claude/skills/
```
