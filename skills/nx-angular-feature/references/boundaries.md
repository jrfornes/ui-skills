# Tags, boundaries, and verification

## Tag taxonomy

Every project gets two tags: one `type:*` and one `scope:*`. This includes apps. An untagged project that matches no constraint is blocked from importing any library once real constraints exist. The error is `A project without tags matching at least one constraint cannot depend on any libraries`.

| `type:` | May depend on |
| --- | --- |
| `app` | `feature`, `ui`, `data-access`, `util` |
| `feature` | `feature`, `ui`, `data-access`, `util` |
| `ui` | `ui`, `util` |
| `data-access` | `data-access`, `util` |
| `util` | `util` |

| `scope:` | May depend on |
| --- | --- |
| `<domain>` (for example `orders`) | the same domain and `shared` |
| `shared` | `shared` only |

Apps usually carry `scope:<app-name>`, and that scope may depend on every domain scope it mounts. Use whatever the workspace already has. Only introduce this scheme if the user agrees.

## Reference `depConstraints`

These go inside `@nx/enforce-module-boundaries` in the root `eslint.config.mjs`, or in `.eslintrc.json` for older workspaces:

```js
depConstraints: [
  { sourceTag: 'type:app', onlyDependOnLibsWithTags: ['type:feature', 'type:ui', 'type:data-access', 'type:util'] },
  { sourceTag: 'type:feature', onlyDependOnLibsWithTags: ['type:feature', 'type:ui', 'type:data-access', 'type:util'] },
  { sourceTag: 'type:ui', onlyDependOnLibsWithTags: ['type:ui', 'type:util'] },
  { sourceTag: 'type:data-access', onlyDependOnLibsWithTags: ['type:data-access', 'type:util'] },
  { sourceTag: 'type:util', onlyDependOnLibsWithTags: ['type:util'] },
  { sourceTag: 'scope:shop', onlyDependOnLibsWithTags: ['scope:orders', 'scope:cart', 'scope:shared'] },
  { sourceTag: 'scope:orders', onlyDependOnLibsWithTags: ['scope:orders', 'scope:shared'] },
  { sourceTag: 'scope:cart', onlyDependOnLibsWithTags: ['scope:cart', 'scope:shared'] },
  { sourceTag: 'scope:shared', onlyDependOnLibsWithTags: ['scope:shared'] },
],
```

When a project has several tags, every matching constraint must be satisfied.

## Verification sequence

Run all of these. Each one catches problems the others miss.

```bash
# 1. Tags are set on every new or moved project
npx nx show project <name> --json   # check "tags"

# 2. Boundary rule and other lint rules
npx nx affected -t lint              # or: npx nx run-many -t lint -p <projects>

# 3. Deep imports through an alias (lint cannot see these, see below)
grep -rnE "from '@[^']+/src/" --include='*.ts' apps libs

# 4. Type-check each touched lib, then build the app
npx nx affected -t typecheck         # only if the workspace defines typecheck targets
npx tsc -p libs/<scope>/<lib>/tsconfig.lib.json --noEmit
npx nx build <app>

# 5. Tests on everything affected
npx nx affected -t test

# 6. Optional visual check of the slice
npx nx graph --focus=<feature-lib> --file=/tmp/graph.html
```

`nx affected` compares against the base branch set in `nx.json` (`defaultBase`). Pass `--base=<ref>` if that is wrong for the current branch.

## What each check catches

| Problem | Lint | Type-check | Grep |
| --- | --- | --- | --- |
| `type:ui` importing a `type:feature` lib through its alias | yes | no | no |
| Domain A importing domain B | yes | no | no |
| Relative import into another project (`../../../orders/...`) | yes (`Projects cannot be imported by a relative or absolute path`) | no | no |
| Deep import through an alias (`@org/orders/feature-list/src/lib/...`) | **no**, even if it also violates a tag rule | yes (`TS2307`) | yes |
| Untagged app or lib | yes | no | no |
| Symbol missing from `index.ts` | no | yes | no |

## Resolving violations

| Violation | Fix |
| --- | --- |
| UI lib needs data | Pass data in with `input()` and emit events with `output()`. The feature lib injects the store and binds it. |
| Two features need the same component | Move it into a `ui` lib: domain-scoped if both features are in the same domain, `shared` otherwise. |
| Feature A needs feature B's data | Import B's `data-access`, not B's feature. If A and B are in different domains, move the shared model or API into `shared/data-access-*`. |
| Data-access needs a formatter from a UI lib | Move the formatter into a `util` lib. |
| Circular dependency between two libs | Extract the shared part into a third, lower-level lib that both import. |

Do not add `// eslint-disable`, widen `allow`, or loosen `depConstraints` to get lint to pass unless the user explicitly requests a rule change.
