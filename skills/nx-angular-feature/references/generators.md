# Generator notes

A list of traps, not commands to paste. Add `--no-interactive` so a missing option fails instead of hanging on a prompt.

## Naming

The positional argument is the project directory. Pass `--name` and `--importPath` following one pattern:

| Directory | `--name` | `--importPath` |
| --- | --- | --- |
| `libs/orders/feature-list` | `orders-feature-list` | `@org/orders/feature-list` |
| `libs/orders/ui` | `orders-ui` | `@org/orders/ui` |
| `libs/orders/data-access` | `orders-data-access` | `@org/orders/data-access` |
| `libs/shared/util-format` | `shared-util-format` | `@org/shared/util-format` |

If the workspace uses another pattern, for example `libs/orders/feature/list` or `@org/orders-feature-list`, copy that pattern instead.

The `--prefix` and `--name` you choose both feed the generated component's selector, so `--name=orders-ui --prefix=orders` yields `orders-orders-ui`. Set the real selector when you replace the placeholder.

## Test runner

The library generator's `--unitTestRunner` default is derived from the Angular version and from whether the library is buildable. It is not inherited from an `@nx/angular:application` default in `nx.json`. On a jest workspace, getting it wrong pulls in a vitest toolchain (`@nx/vitest`, `@nx/vite`, AnalogJS plugins, `jsdom`, a root `vitest.config.mts`).

## When a generator fails partway

The library generator writes its files and then runs the package manager install. If the install fails, the files are already on disk and the project is already registered in `tsconfig.base.json` and `nx.json` — the generator is not transactional.

Do not re-run it; that will collide with what it already created. Instead fix the dependency problem and install again, or revert the whole change with git and start over. `--skipPackageJson` avoids the install entirely when you would rather add dependencies yourself.

## What each library type leaves to clean up

Every `@nx/angular:library` invocation creates a placeholder standalone component at `src/lib/<name>/<name>.ts` and exports it from `src/index.ts`. This is not affected by `--skipModule`, which only applies to module-based (`--standalone=false`) libraries.

**Feature library** (`--routing --lazy --parent=...`) additionally creates `src/lib/lib.routes.ts` exporting `<camelName>Routes`, and updates the parent route file with a `loadChildren` entry. Both the URL segment and the exported symbol come from the **project name**, not the directory, so `libs/orders/feature-list` named `orders-feature-list` produces `path: 'orders-feature-list'` and `ordersFeatureListRoutes`. Then:

1. Rename the route path in the parent file to the real URL segment, for example `orders`.
2. Rename the routes export to something meaningful, such as `ordersRoutes`, and update the parent `import(...)` call to match.
3. Rename the placeholder component to the real page, or replace it.
4. Remove the component export from `index.ts`, leaving only the routes.

To mount a feature inside another feature, set `--parent` to the parent feature's `lib.routes.ts`. `--lazy=false` adds the routes as `children` instead of `loadChildren`; only use it when the workspace deliberately avoids lazy loading.

**UI library**: delete or rename the placeholder, then add real components with `npx nx g @nx/angular:component <path-under-src/lib> --export`. `--export` adds the component to `index.ts`; leave it off for internal sub-components.

**Data-access library**: delete `src/lib/<name>/` and its export line — data-access libs should contain no components. Add services with `@nx/angular:service`, whose `--path` is relative to the workspace root, not the project. If the workspace uses NgRx, check `npx nx list @nx/angular` for `ngrx-feature-store` and follow the store pattern already used in the repo.

**Util library**: delete the placeholder the same way. If the util has no Angular dependency at all and the workspace has `@nx/js`, `npx nx g @nx/js:library` produces a lighter lib with no placeholder component. Only use it if the repo already does that.

## Buildable and publishable libs

Only pass `--buildable` or `--publishable` if neighbouring libs in the workspace already use them. They add build targets, change the default test runner, and make `enforceBuildableLibDependency` apply: a buildable lib may not import a non-buildable one.

## Moving or renaming

Use generators, not `mv`, so aliases, tsconfig paths, and imports are rewritten:

```bash
npx nx g @nx/workspace:move --project=orders-ui --destination=libs/shared/ui-order-card \
  --newProjectName=shared-ui-order-card --importPath=@org/shared/ui-order-card --dry-run
npx nx g @nx/workspace:remove --projectName=<name>
```

The move carries the old tags across unchanged, so update them afterwards: a lib moved into `shared` needs `scope:shared`.

## Deprecated executors

Recent Nx versions emit warnings when a generator writes an executor-based `lint` or `test` target, pointing at `nx g @nx/eslint:convert-to-inferred` and `nx g @nx/vitest:convert-to-inferred`. Leave the migration to the workspace owners unless asked, but expect the warnings.
