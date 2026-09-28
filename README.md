# ui-skills

Agent skills for UI work. Each skill is a folder under `skills/` containing a `SKILL.md` with frontmatter (`name`, `description`) and optional `references/` files that the agent loads on demand.

| Skill | Use it for |
| --- | --- |
| [`nx-angular-feature`](skills/nx-angular-feature/SKILL.md) | Building Angular features in an Nx monorepo as a vertical slice across feature, ui, data-access, and util libs, with lazy routing, tight public APIs, and module boundaries that still pass |

## Installing

Copy or symlink a skill folder into your agent's skills directory, for example:

```bash
# Cursor (project-level)
mkdir -p .cursor/skills && cp -r skills/nx-angular-feature .cursor/skills/

# Claude Code (project-level)
mkdir -p .claude/skills && cp -r skills/nx-angular-feature .claude/skills/
```
