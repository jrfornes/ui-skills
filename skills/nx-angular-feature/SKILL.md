---
name: nx-angular-feature
description: Build an Angular feature in an Nx monorepo as a vertical slice across libraries (feature / ui / data-access / util) instead of dumping code into the app shell or inventing folders. Use when adding or changing a page, route, screen, component, service, store, or shared UI in an Nx workspace with Angular (nx.json plus @nx/angular), or when the user mentions Nx libs, generators, lazy routes, tags, or module boundaries.
---

# nx-angular-feature

Deliver Angular features as a vertical slice through the Nx project graph. The work is not done until the code sits in the right library, is reachable through lazy routing, is exported only through a tight public API, and the boundary rules still pass.

Work through these five steps in order. Do not skip step 1: most bad placements come from writing code before deciding where it belongs.

```
- [ ] 1. Discover workspace conventions
- [ ] 2. Classify every piece of the change
- [ ] 3. Place files with generators (dry run first)
- [ ] 4. Wire routing and lazy loading
- [ ] 5. Tighten public APIs, then verify boundaries
```

## 1. Discover workspace conventions

Read these before creating anything. The workspace's existing conventions win over the defaults in this skill.

| Look at | To learn |
| --- | --- |
| `nx.json` (`generators` block) | Defaults the team already set for generators (style, test runner, prefix, standalone) |
| `tsconfig.base.json` `compilerOptions.paths` | Import alias format, for example `@org/orders/feature-list` or `@org/orders-feature-list` |
| `ls libs/` (or `packages/`) and a couple of `project.json` files | Folder layout, project naming, and the tags already in use |
| Root `eslint.config.*` or `.eslintrc.json`, rule `@nx/enforce-module-boundaries` | The `depConstraints` that define what can import what |
| `apps/<app>/src/app/app.routes.ts` and `app.config.ts` | How features are mounted and where global providers live |

Useful commands:

```bash
npx nx show projects                      # all projects
npx nx show project <name> --json         # root, tags, targets for one project
npx nx list @nx/angular                   # generators installed in this workspace
npx nx g @nx/angular:library --help       # flags for THIS Nx version
```

Always check `--help` for the installed version. Generator flags change between Nx majors.

If the workspace has no tags or only `sourceTag: '*'` constraints, keep the taxonomy below in mind, ask before introducing tags workspace-wide, and still place code as if the rules existed.

## 2. Classify every piece of the change

Split the request into pieces and give each piece exactly one home. One feature usually touches several library types.

| Piece of the change | Home | Default tag |
| --- | --- | --- |
| Top-level layout, top-level route table, `bootstrapApplication` providers, environment config | App shell: `apps/<app>` | `type:app` |
| Routed page or container: reads state, calls services, composes UI | Feature lib: `libs/<scope>/feature-<name>` | `type:feature` |
| Presentational component: only `input()` / `output()`, no injected services that fetch or mutate state | UI lib: `libs/<scope>/ui` or `libs/shared/ui-<name>` | `type:ui` |
| HTTP clients, stores or signals state, facades, resolvers, guards that read state, domain models | Data-access lib: `libs/<scope>/data-access` | `type:data-access` |
| Pure functions, pipes, validators, constants, types with no Angular DI | Util lib: `libs/<scope>/util-<name>` | `type:util` |

Scope rules:

- Used by one domain: put it in that domain's scope (`scope:orders`).
- Needed by two or more domains: put it in `libs/shared/...` with `scope:shared`. Move it there when the second consumer appears, not before.
- A feature never imports another domain's feature or data-access library. If it needs to, the shared part belongs in `shared`.

App-shell test: if deleting the feature would require editing more than the one route entry and possibly one nav link in `apps/<app>`, too much of it lives in the app. Components, services, and state for a feature do not belong under `apps/`.

Before placing anything, reuse what exists: search for an existing lib with the same scope and type (`npx nx show projects | grep <scope>`) and add to it rather than creating a sibling.

## 3. Place files with generators

Use Nx generators rather than creating project folders by hand. Generators register the project, add the path alias, create lint and test config, and set tags. Run with `--dry-run` first, read the planned file list, then run for real.

```bash
npx nx g @nx/angular:library libs/orders/feature-list \
  --name=orders-feature-list \
  --importPath=@org/orders/feature-list \
  --tags=scope:orders,type:feature \
  --prefix=orders \
  --routing --lazy --parent=apps/shop/src/app/app.routes.ts \
  --dry-run
```

Always pass `--name` and `--importPath` explicitly, following the convention from step 1. Without them Nx derives both from the last folder name only: `libs/orders/ui` becomes project `ui` with alias `@org/ui`, and the next `libs/cart/ui` fails with a name collision.

Components, services, and other building blocks go inside an existing lib, using a path under that lib's `src/lib/`:

```bash
npx nx g @nx/angular:component libs/orders/ui/src/lib/order-card/order-card --export --dry-run
npx nx g @nx/angular:service orders-api --project=orders-data-access \
  --path=libs/orders/data-access/src/lib/orders-api --dry-run
```

For every library type, the exact commands and the generated placeholder files to clean up afterward are in [references/generators.md](references/generators.md).

Never:

- create `libs/**` folders, `project.json`, or `tsconfig` files by hand;
- put feature components under `apps/<app>/src/app/<feature>/`;
- invent parallel folder schemes (`src/features/`, `src/modules/`, `libs/common/`) that the workspace does not already use.

## 4. Wire routing and lazy loading

Every feature lib exports a route array, and the app mounts it lazily. Nothing in the app eagerly imports feature components.

With `--routing --lazy --parent=...` the generator adds the lazy route for you. Afterward, rename the generated path and export to meaningful names:

```ts
// apps/shop/src/app/app.routes.ts
export const appRoutes: Route[] = [
  {
    path: 'orders',
    loadChildren: () =>
      import('@org/orders/feature-list').then((m) => m.ordersRoutes),
  },
];
```

```ts
// libs/orders/feature-list/src/lib/lib.routes.ts
export const ordersRoutes: Route[] = [
  {
    path: '',
    providers: [OrdersStore],
    children: [
      { path: '', component: OrderListPage },
      { path: ':id', loadComponent: () => import('./order-detail/order-detail-page').then((m) => m.OrderDetailPage) },
    ],
  },
];
```

Rules:

- The app shell references features only through `loadChildren` or `loadComponent` with the library alias. A static `import { X } from '@org/orders/feature-list'` in the app pulls the feature into the main bundle.
- Feature-scoped providers (stores, facades) go in the route's `providers`, not in `app.config.ts`. Only truly global providers (`provideRouter`, `provideHttpClient`, interceptors, app-wide auth) belong in `app.config.ts`.
- Nested features mount the same way from the parent feature's route file, not from the app.
- Keep guards and resolvers next to the data they read: in data-access, or in the feature lib if they only apply to its routes.

## 5. Tighten public APIs, then verify boundaries

`src/index.ts` is the library's entire contract. Export only what consumers need:

| Lib type | Exports from `index.ts` |
| --- | --- |
| feature | The routes array only, for example `export { ordersRoutes } from './lib/lib.routes';` |
| ui | The components and directives other libs render, plus their input types |
| data-access | Facade or store, public models, and `provide*()` functions. Not raw HTTP DTOs or internal effects |
| util | The functions, pipes, and types intended for reuse |

The feature-lib generator also exports the placeholder component (`export * from './lib/<name>/<name>'`). Remove that line. Prefer named exports over `export *` so every addition to the API is deliberate.

Consumers import only through the alias root (`@org/orders/ui`). Never import through `@org/orders/ui/src/...` or relative paths into another project.

Then run the verification in [references/boundaries.md](references/boundaries.md). In short:

```bash
npx nx affected -t lint test build                      # or: nx run-many -t lint test -p <touched projects>
npx tsc -p libs/<scope>/<lib>/tsconfig.lib.json --noEmit # each touched lib, unless a typecheck target exists
grep -rnE "from '@[^']+/src/" --include='*.ts' apps libs # deep imports through an alias
```

Lint alone is not proof. A deep import such as `@org/orders/feature-list/src/lib/...` cannot be resolved by the boundaries rule, so ESLint reports nothing, even when the import also breaks a tag constraint. Only a type-check catches it. Angular libs generated by Nx often have no `typecheck` target, and the app `build` only compiles libs the app actually imports, so type-check each touched lib directly.

## Definition of done

- [ ] Each new piece lives in the library type chosen in step 2. The app shell only gained a route entry and possibly a nav link.
- [ ] New libs were created by generators with explicit `--name`, `--importPath`, and `scope:*` plus `type:*` tags that match the workspace convention.
- [ ] Generator placeholder components that serve no purpose have been deleted, and so have their exports.
- [ ] The feature is reachable through `loadChildren` or `loadComponent`, with no eager imports of feature code from the app.
- [ ] Each touched `index.ts` exports only the public surface described above.
- [ ] `lint` passes on every touched project with the `@nx/enforce-module-boundaries` rule active, and nothing was added to `allow`, `depConstraints`, or `eslint-disable` to make it pass.
- [ ] Every touched lib type-checks, the app builds, and the deep-import grep returns nothing.

If a boundary rule blocks the design, change the design: move the shared piece down into `ui`, `data-access`, `util`, or `shared`. Do not loosen the rule unless the user explicitly asks for that.
