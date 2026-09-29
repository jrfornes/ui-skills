---
name: nx-angular-feature
description: Build an Angular feature in an Nx monorepo as a vertical slice across libraries (feature / ui / data-access / util) instead of dumping code into the app shell or inventing folders. Use when adding or changing a page, route, screen, component, service, store, or shared UI in an Nx workspace with Angular (nx.json plus @nx/angular), or when the user mentions Nx libs, generators, lazy routes, tags, or module boundaries.
---

# nx-angular-feature

Deliver Angular features as a vertical slice through the Nx project graph. The work is not done until the code sits in the right library, is reachable through lazy routing, is exported only through a tight public API, and the boundary rules still pass.

This skill owns **where code goes** and **what must be true when you are done**, not the Nx CLI surface. Flags, defaults, and generated file names change between versions, so confirm every one against the installed version before using it. Sources, in order:

1. Nx's own agent skills (`nx-generate`, `nx-workspace`), if installed.
2. Nx MCP tools, if present. The MCP server ships separately from Nx, so it can lag the installed version or fail; fall back and keep going.
3. `--help`, always available: `npx nx g @nx/angular:library --help`.

**Scope check.** Adding a field, fixing a template, or changing a method body needs none of this: work in place and run that project's checks. Follow the steps below when the change introduces a new page, route, screen, store, or shared component, or when you are about to create a file and do not know which library it belongs in. Do not skip step 1: most bad placements come from writing code before deciding where it belongs.

## 1. Discover workspace conventions

The workspace's existing conventions win over the defaults in this skill.

| Look at | To learn |
| --- | --- |
| `nx.json` (`generators` block) | Generator defaults the team already set. Check which generator each default applies to: defaults for `@nx/angular:application` say nothing about `@nx/angular:library` |
| `tsconfig.base.json` `compilerOptions.paths` | Import alias format, for example `@org/orders/feature-list` or `@org/orders-feature-list` |
| `nx show projects`, `nx show project <name> --json` | Project names, roots, tags, and targets |
| `@nx/enforce-module-boundaries` in the root ESLint config | The `depConstraints` that define what can import what |
| `apps/<app>/src/app/app.routes.ts` and `app.config.ts` | How features are mounted and where global providers live |

Use `nx show project <name> --json` rather than reading `project.json`: targets inferred by plugins do not appear in the file, so it can suggest a project has no `test` or `typecheck` target when it does.

If the workspace has no tags, or only `sourceTag: '*'` constraints, place code as if the taxonomy below were enforced, but do not roll out a tag scheme across existing projects without asking.

## 2. Classify every piece of the change

Give each piece of the request exactly one home. One feature usually touches several library types.

| Piece of the change | Home | Default tag |
| --- | --- | --- |
| Top-level layout, top-level route table, `bootstrapApplication` providers, environment config | App shell: `apps/<app>` | `type:app` |
| Routed page or container: reads state, calls services, composes UI | Feature lib: `libs/<scope>/feature-<name>` | `type:feature` |
| Presentational component: only `input()` / `output()`, no injected services that fetch or mutate state | UI lib: `libs/<scope>/ui` or `libs/shared/ui-<name>` | `type:ui` |
| HTTP clients, stores or signals state, facades, resolvers, guards that read state, domain models | Data-access lib: `libs/<scope>/data-access` | `type:data-access` |
| Pure functions, pipes, validators, constants, types with no Angular DI | Util lib: `libs/<scope>/util-<name>` | `type:util` |

- Used by one domain: put it in that domain's scope (`scope:orders`).
- Needed by two or more domains: move it to `libs/shared/...` with `scope:shared` when the second consumer appears, not before.
- A feature never imports another domain's feature or data-access library. If it needs to, the shared part belongs in `shared`.

App-shell test: if deleting the feature would require editing more than one route entry and possibly one nav link in `apps/<app>`, too much of it lives in the app.

Reuse before creating: search for an existing lib with the same scope and type (`npx nx show projects -p 'tag:scope:orders'`) and add to it. A new library is justified only when it gives code a different dependency rule than its neighbours — a different `type:`, a different `scope:`, or a lazy boundary. "This is a new page" is not enough: several routed pages in one domain can share a single `feature` lib until they need to load independently.

## 3. Place files with generators

Generators register the project, add the path alias, create lint and test config, and set tags, so never create `libs/**` folders, `project.json`, or `tsconfig` files by hand. Look up the options for the installed version, then adapt this shape rather than pasting it:

```bash
npx nx g @nx/angular:library libs/orders/feature-list \
  --name=orders-feature-list \
  --importPath=@org/orders/feature-list \
  --tags=scope:orders,type:feature \
  --prefix=orders \
  --unitTestRunner=<whatever the rest of the workspace uses> \
  --routing --lazy --parent=apps/shop/src/app/app.routes.ts \
  --no-interactive
```

Easy to get wrong and expensive to undo:

- **Always pass `--name` and `--importPath`.** Nx derives both from the last path segment only, so `libs/orders/ui` becomes project `ui` with alias `@<workspace-name>/ui`, and the next `libs/cart/ui` collides.
- **Always pass `--unitTestRunner`**, matching the workspace. The library generator's default is version-dependent and differs from the application generator's, so omitting it can add a second test framework.
- **`--dry-run` is not universal.** It works for `@nx/angular:component`, `@nx/angular:service`, and `@nx/workspace:move`, but generators that install packages, including `@nx/angular:library`, reject it. Where you cannot dry-run, confirm the options first.

Components and services go inside an existing lib, under its `src/lib/`:

```bash
npx nx g @nx/angular:component libs/orders/ui/src/lib/order-card/order-card --export --dry-run
npx nx g @nx/angular:service orders-api --project=orders-data-access \
  --path=libs/orders/data-access/src/lib/orders-api --dry-run
```

Do not put feature components under `apps/<app>/src/app/<feature>/` or invent folder schemes (`src/features/`, `src/modules/`, `libs/common/`) the workspace does not already use. For generator placeholder files and recovery from a partial failure, see [references/generators.md](references/generators.md).

## 4. Wire routing and lazy loading

With `--routing --lazy --parent=...` the generator adds the lazy route, deriving the URL segment and exported symbol from the project name. Rename both to something meaningful:

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

- The app shell references features only through `loadChildren` or `loadComponent` with the library alias. A static import from the feature pulls it into the main bundle.
- Feature-scoped providers (stores, facades) go in the route's `providers`. Only global providers (`provideRouter`, `provideHttpClient`, interceptors, app-wide auth) belong in `app.config.ts`.
- Nested features mount from the parent feature's route file, not from the app.
- Keep guards and resolvers next to the data they read: in data-access, or in the feature lib if they only apply to its routes.

## 5. Tighten public APIs, then verify

`src/index.ts` is the library's entire contract. Export only what consumers need:

| Lib type | Exports from `index.ts` |
| --- | --- |
| feature | The routes array. If the app mounts a single page with `loadComponent` from the alias, that one page component instead |
| ui | The components and directives other libs render, plus their input types |
| data-access | Facade or store, public models, and `provide*()` functions. Not raw HTTP DTOs or internal effects |
| util | The functions, pipes, and types intended for reuse |

Remove the generator's placeholder component export. Prefer named exports over `export *` so every addition to the API is deliberate. Consumers import only through the alias root (`@org/orders/ui`), never through `@org/orders/ui/src/...` or relative paths into another project.

### Verify

No single check covers this. Run all four; [references/boundaries.md](references/boundaries.md) explains what each catches and misses.

```bash
# 1. Boundaries and lint, on the projects you actually touched
npx nx run-many -t lint test -p <touched projects>

# 2. Type-check each touched lib (catches deep imports that lint cannot see)
npx tsc -p libs/<scope>/<lib>/tsconfig.lib.json --noEmit   # unless the project has a typecheck target

# 3. Compile the app: the only check that type-checks Angular templates
npx nx build <app>

# 4. Deep imports through an alias
grep -rnE "from '@[^']+/src/" --include='*.ts' <project root dirs>
```

- **`nx affected` can pass without running anything.** Once your work is committed on the base branch, it prints `No tasks were run` and exits 0. Use `run-many` on touched projects first; treat `affected` with an explicit `--base` as a wider secondary sweep.
- **Only the Angular compiler checks templates.** Lint and `tsc` both pass a template that references a missing property. `nx build <app>` sees lazily loaded libs through `loadChildren`, so a feature wired per step 4 is covered, but a `ui` lib with no consumer yet is not.
- **The grep is only as good as its directories.** Point it at the workspace's real project roots. On a directory that does not exist it prints only to stderr, which looks exactly like success.

## Definition of done

- [ ] The app shell only gained a route entry and possibly a nav link; everything else lives in the lib type chosen in step 2.
- [ ] New projects carry the same kind of tags as their neighbours. If the workspace has no tag scheme, do not invent one; raise it with the user.
- [ ] Every touched `index.ts` exports only the public surface, with placeholders removed.
- [ ] `lint` passes with `@nx/enforce-module-boundaries` active, and nothing was added to `allow`, `depConstraints`, or `eslint-disable` to make it pass. If the constraints are only `sourceTag: '*'`, say so rather than reporting that boundaries were enforced.
- [ ] Every touched lib type-checks, the app builds, and the deep-import grep returns nothing.

If a boundary rule blocks the design, change the design: move the shared piece down into `ui`, `data-access`, `util`, or `shared`. Do not loosen the rule unless the user explicitly asks.
