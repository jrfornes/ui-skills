# Tags, boundaries, and verification

## Tag taxonomy

Every project gets one `type:*` and one `scope:*` tag, including apps. Apps usually carry `scope:<app-name>`, which may depend on every domain scope it mounts. Use whatever the workspace already has; only introduce this scheme if the user agrees.

Once real constraints exist, an untagged project cannot import any library: `A project without tags matching at least one constraint cannot depend on any libraries`. Nx scaffolds applications with `tags: []`, so adopting the constraints breaks the app's lint, reported on its route file. Tag the app in the same change.

The reference constraints go inside `@nx/enforce-module-boundaries` in the root `eslint.config.mjs`, or `.eslintrc.json` in older workspaces:

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

## Checks beyond the four in SKILL.md

```bash
npx nx show project <name> --json                          # "tags" set on every new or moved project
npx nx affected -t lint test build --base=origin/main      # wider sweep; without --base it can silently run nothing
npx nx graph --focus=<feature-lib> --file=/tmp/graph.html  # optional visual check of the slice
```

## What each check catches

| Problem | Lint | Type-check | App build | Grep |
| --- | --- | --- | --- | --- |
| `type:ui` importing a `type:feature` lib through its alias | yes | no | no | no |
| Domain A importing domain B | yes | no | no | no |
| Relative import into another project (`../../../orders/...`) | yes (`Projects cannot be imported by a relative or absolute path, and must begin with a npm scope`) | no | no | no |
| Deep import through an alias (`@org/orders/feature-list/src/lib/...`) | **no**, even if it also violates a tag rule | yes (`TS2307`) | only if the app reaches the lib | yes |
| Untagged app or lib | yes | no | no | no |
| Symbol missing from `index.ts` | no | yes | yes | no |
| Angular template referencing a member that does not exist | **no** | **no** | yes, for libs the app reaches | no |

Deep imports are invisible to the boundaries rule because it resolves an import to a project before checking constraints and has no message for deep imports, so an unresolvable specifier is silently skipped.

A lib with its own `build` target (buildable or publishable) can have its templates checked by running that target, even before the app consumes it.

## Resolving violations

| Violation | Fix |
| --- | --- |
| UI lib needs data | Pass data in with `input()` and emit events with `output()`. The feature lib injects the store and binds it. |
| Two features need the same component | Move it into a `ui` lib: domain-scoped if both features are in the same domain, `shared` otherwise. |
| Feature A needs feature B's data | Import B's `data-access`, not B's feature. If A and B are in different domains, move the shared model or API into `shared/data-access-*`. |
| Data-access needs a formatter from a UI lib | Move the formatter into a `util` lib. |
| Circular dependency between two libs | Extract the shared part into a third, lower-level lib that both import. |
