# ui-skills

Agent skills for UI work. Each skill is a folder under `skills/` containing a `SKILL.md` with frontmatter (`name`, `description`) and optional `references/` files that the agent loads on demand.

| Skill | Use it for |
| --- | --- |
| [`ngrx-store-change`](skills/ngrx-store-change/SKILL.md) | Changing NgRx state as a complete unit — actions, effects with a cancel and error path, immutable reducers, selectors, and a facade only where the workspace already has them |

## Installing

Copy or symlink a skill folder into your agent's skills directory, for example:

```bash
# Cursor (project-level)
mkdir -p .cursor/skills && cp -r skills/ngrx-store-change .cursor/skills/

# Claude Code (project-level)
mkdir -p .claude/skills && cp -r skills/ngrx-store-change .claude/skills/
```
