# Generator recipes

These recipes were verified against Nx 23 (`@nx/angular` 23.x). Run `npx nx g <generator> --help` to confirm flags on older or newer versions. Replace `@org`, the scope, and the app path with the workspace's own conventions.

Always run with `--dry-run` first. Add `--no-interactive` when running non-interactively so a missing option fails instead of hanging on a prompt.

## Naming

The positional argument is the project directory. Nx derives the project name and the import path from the **last path segment only**, so always pass both explicitly:

| Directory | `--name` | `--importPath` |
| --- | --- | --- |
| `libs/orders/feature-list` | `orders-feature-list` | `@org/orders/feature-list` |
| `libs/orders/ui` | `orders-ui` | `@org/orders/ui` |
| `libs/orders/data-access` | `orders-data-access` | `@org/orders/data-access` |
| `libs/shared/util-format` | `shared-util-format` | `@org/shared/util-format` |

If the workspace uses another pattern, for example `libs/orders/feature/list` or `@org/orders-feature-list`, copy that pattern instead.

## Feature library (routed)

```bash
npx nx g @nx/angular:library libs/orders/feature-list \
  --name=orders-feature-list --importPath=@org/orders/feature-list \
  --tags=scope:orders,type:feature --prefix=orders \
  --routing --lazy --parent=apps/shop/src/app/app.routes.ts
```

What it generates:

- `src/lib/lib.routes.ts` with `export const <camelName>Routes: Route[]`.
- A placeholder component at `src/lib/<name>/<name>.ts`.
- An `index.ts` that exports both the routes and the placeholder component.
- An `UPDATE` to the parent route file, adding `loadChildren: () => import('<importPath>').then((m) => m.<camelName>Routes)` under a path equal to the directory's last segment.

Cleanup:

1. Rename the route path in the parent file to the real URL segment, for example `orders`.
2. Rename the routes export to something meaningful, such as `ordersRoutes`, and update the parent `import(...)` call to match.
3. Rename the placeholder component to the real page, or replace it.
4. Remove the component export from `index.ts`, leaving only the routes.

To mount a feature inside another feature, set `--parent` to the parent feature's `lib.routes.ts`. `--lazy=false` adds the routes as `children` instead of `loadChildren`. Only use it when the workspace deliberately avoids lazy loading.

## UI library

```bash
npx nx g @nx/angular:library libs/orders/ui \
  --name=orders-ui --importPath=@org/orders/ui \
  --tags=scope:orders,type:ui --prefix=orders
```

The generator creates a placeholder component `src/lib/<name>/<name>.ts` and exports it. Delete or rename it, then add real components with:

```bash
npx nx g @nx/angular:component libs/orders/ui/src/lib/order-card/order-card --export
```

`--export` adds the component to the lib's `index.ts`. Leave it off for internal sub-components.

## Data-access library

```bash
npx nx g @nx/angular:library libs/orders/data-access \
  --name=orders-data-access --importPath=@org/orders/data-access \
  --tags=scope:orders,type:data-access
```

The standalone library generator still creates a placeholder component, even with `--skipModule`. Data-access libs should contain no components, so delete `src/lib/<name>/` and its export line.

Add services with:

```bash
npx nx g @nx/angular:service orders-api --project=orders-data-access \
  --path=libs/orders/data-access/src/lib/orders-api
```

`--path` is relative to the workspace root, not the project. If the workspace uses NgRx, check `npx nx list @nx/angular` for `ngrx-feature-store` and follow the store pattern already used in the repo.

## Util library

```bash
npx nx g @nx/angular:library libs/shared/util-format \
  --name=shared-util-format --importPath=@org/shared/util-format \
  --tags=scope:shared,type:util
```

Delete the placeholder component, and its export, the same way as for data-access. If the util has no Angular dependency at all and the workspace has `@nx/js`, `npx nx g @nx/js:library` produces a lighter lib. Only use it if the repo already does that.

## Buildable and publishable libs

Only pass `--buildable` or `--publishable` if neighbouring libs in the workspace already use them. They add build targets and make `enforceBuildableLibDependency` apply: a buildable lib may not import a non-buildable one.

## Moving or renaming

Use generators, not `mv`, so aliases, tsconfig paths, and imports are rewritten:

```bash
npx nx g @nx/workspace:move --project=orders-ui --destination=libs/shared/ui-order-card \
  --newProjectName=shared-ui-order-card --importPath=@org/shared/ui-order-card
npx nx g @nx/workspace:remove --projectName=<name>
```

After a move, update the tags in the moved `project.json`. For example, a lib moved into `shared` needs `scope:shared`.
