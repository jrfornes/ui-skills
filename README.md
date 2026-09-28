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

## Pairing with Nx's own agent tooling

`nx-angular-feature` is an architecture skill: it decides which library a piece of an Angular feature belongs in and what has to be true before the work is done. It deliberately does not duplicate the Nx CLI surface, because flags and generator defaults change between versions.

Nx ships its own agent configuration for the mechanical half. `nx configure-ai-agents` registers the Nx MCP server, writes `AGENTS.md`/`CLAUDE.md` rules, and installs skills including `nx-workspace` (project and target discovery) and `nx-generate` (generator discovery and execution). Install those alongside this skill and they compose: Nx's skills answer "what does this generator do in this workspace", this one answers "where does the code go".

Everything here still works without them — the skill falls back to `nx <command> --help`, which matters because the MCP server is distributed separately from Nx and can lag the installed version.
