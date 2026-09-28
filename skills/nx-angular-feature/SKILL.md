---
name: nx-angular-feature
description: Build an Angular feature in an Nx monorepo as a vertical slice across libraries (feature / ui / data-access / util) instead of dumping code into the app shell or inventing folders. Use when adding or changing a page, route, screen, component, service, store, or shared UI in an Nx workspace with Angular (nx.json plus @nx/angular), or when the user mentions Nx libs, generators, lazy routes, tags, or module boundaries.
---

# nx-angular-feature

Deliver Angular features as a vertical slice through the Nx project graph. The work is not done until the code sits in the right library, is reachable through lazy routing, is exported only through a tight public API, and the boundary rules still pass.

This skill owns **where code goes** and **what must be true when you are done**. It does not own the Nx CLI surface: flags, defaults, and generated file names change between versions, so look those up at runtime instead of trusting any command written here.

## Use the Nx tooling for discovery and execution

Prefer these sources, in order, for anything version-specific:

1. **Nx's own agent skills**, if present: `nx-generate` (generator discovery and execution) and `nx-workspace` (projects, targets, configuration). `nx configure-ai-agents` installs them.
2. **Nx MCP tools**, if present: `nx_generators`, `nx_generator_schema`, `nx_workspace`, `nx_project_details`, `nx_docs`.
3. **`--help`**, always available: `npx nx g @nx/angular:library --help`.

Treat the first two as optional. The MCP server ships separately from Nx (`nx mcp` fetches the `nx-mcp` package), so it can lag the installed Nx version and some tools may error or be disabled; `nx_docs` is a remote service call. When a tool is missing or fails, fall back to `--help` and keep going.

Never copy a flag or a default out of this skill into a command without confirming it against one of those three sources for the installed version.

## Scope check

Adding a field to an existing component, fixing a template, or changing a service's method body needs none of this. Work in place and run that project's checks.

Use the five steps below when the change introduces a new page, route, screen, store, or shared component, or when you are about to create a file and do not already know which library it belongs in.

```
- [ ] 1. Discover workspace conventions
- [ ] 2. Classify every piece of the change
- [ ] 3. Place files with generators
- [ ] 4. Wire routing and lazy loading
- [ ] 5. Tighten public APIs, then verify
```

Do not skip step 1: most bad placements come from writing code before deciding where it belongs.

## 1. Discover workspace conventions

The workspace's existing conventions win over the defaults in this skill.

| Look at | To learn |
| --- | --- |
| `nx.json` (`generators` block) | Generator defaults the team already set. Read which generator each default applies to: a block that only configures `@nx/angular:application` tells you nothing about what `@nx/angular:library` will do |
| `tsconfig.base.json` `compilerOptions.paths` | Import alias format, for example `@org/orders/feature-list` or `@org/orders-feature-list` |
| `nx show projects` and `nx show project <name> --json` | Project names, roots, tags, and targets |
| Root `eslint.config.*` or `.eslintrc.json`, rule `@nx/enforce-module-boundaries` | The `depConstraints` that define what can import what |
| `apps/<app>/src/app/app.routes.ts` and `app.config.ts` | How features are mounted and where global providers live |

Useful commands:

```bash
npx nx show projects                              # all projects
npx nx show projects -p 'tag:scope:orders'        # projects in one scope
npx nx show project <name> --json                 # root, tags, and targets for one project
npx nx list @nx/angular                           # generators installed in this workspace
```

Use `nx show project <name> --json` rather than reading `project.json`: targets inferred by plugins do not appear in the file, so the file alone can tell you a project has no `test` or `typecheck` target when it does.

If the workspace has no tags, or only `sourceTag: '*'` constraints, keep the taxonomy below in mind and place code as if the rules existed, but do not roll out a tag scheme across existing projects without asking. See the note in step 5 about what that means for the definition of done.

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

Reuse before creating. Search for an existing lib with the same scope and type (`npx nx show projects -p 'tag:scope:orders'`) and add to it. A new library is justified when it gives a piece of code a different dependency rule than its neighbours — a different `type:`, a different `scope:`, or a lazy boundary. It is not justified by "this is a new page": one lib per page produces a graph nobody can reason about. Several routed pages in one domain can share a single `feature` lib until they need to load independently.

## 3. Place files with generators

Use Nx generators rather than creating project folders by hand. Generators register the project, add the path alias, create lint and test config, and set tags.

Look up the options for the installed version first (see the delegation section above), then treat the examples below as shapes to adapt, not as commands to paste.

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

Three things that are easy to get wrong and expensive to undo:

- **Always pass `--name` and `--importPath`.** Nx derives both from the last path segment only, so `libs/orders/ui` becomes project `ui` with alias `@<workspace-name>/ui`, and the next `libs/cart/ui` fails with a name collision.
- **Always pass `--unitTestRunner` explicitly**, set to whatever the workspace already uses. The library generator's default is version- and Angular-dependent and is not the same as the application generator's, so leaving it off can add a second test framework and its whole dependency tree to a workspace that already had one.
- **`--dry-run` is not supported by every generator.** It works for `@nx/angular:component`, `@nx/angular:service`, and `@nx/workspace:move`. Generators that install packages reject it — `@nx/angular:library` currently answers `This generator does not support --dry-run`. Dry-run where you can; where you cannot, confirm the options first and then run for real.

Components, services, and other building blocks go inside an existing lib, using a path under that lib's `src/lib/`:

```bash
npx nx g @nx/angular:component libs/orders/ui/src/lib/order-card/order-card --export --dry-run
npx nx g @nx/angular:service orders-api --project=orders-data-access \
  --path=libs/orders/data-access/src/lib/orders-api --dry-run
```

For the placeholder files each library type leaves behind, and how to recover when a generator fails partway, see [references/generators.md](references/generators.md).

Never:

- create `libs/**` folders, `project.json`, or `tsconfig` files by hand;
- put feature components under `apps/<app>/src/app/<feature>/`;
- invent parallel folder schemes (`src/features/`, `src/modules/`, `libs/common/`) that the workspace does not already use.

## 4. Wire routing and lazy loading

Features are mounted lazily and nothing in the app eagerly imports feature components.

With `--routing --lazy --parent=...` the generator adds the lazy route for you, deriving both the URL segment and the exported symbol from the project name. Rename both to something meaningful:

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

Wiring the feature into the route graph is also what makes it verifiable — see step 5.

## 5. Tighten public APIs, then verify

`src/index.ts` is the library's entire contract. Export only what consumers need:

| Lib type | Exports from `index.ts` |
| --- | --- |
| feature | The routes array, for example `export { ordersRoutes } from './lib/lib.routes';`. Nothing else, unless the app mounts a single page with `loadComponent` from the alias, in which case export that one page component and no other internals |
| ui | The components and directives other libs render, plus their input types |
| data-access | Facade or store, public models, and `provide*()` functions. Not raw HTTP DTOs or internal effects |
| util | The functions, pipes, and types intended for reuse |

The library generator also exports its placeholder component. Remove that line. Prefer named exports over `export *` so every addition to the API is deliberate.

Consumers import only through the alias root (`@org/orders/ui`). Never import through `@org/orders/ui/src/...` or relative paths into another project.

### Verify

No single check covers this. Run all four, and read [references/boundaries.md](references/boundaries.md) for what each one catches and misses.

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

Three traps in this step:

- **`nx affected` can pass without running anything.** Once your work is committed on the base branch, `nx affected -t lint test build` prints `No tasks were run` and exits 0. Use `run-many` with the projects you touched as the primary check, and treat `affected` (with an explicit `--base`) as a wider secondary sweep.
- **Neither lint nor `tsc` type-checks Angular templates.** A template referencing a property that does not exist passes both. Only the Angular compiler catches it, which means `nx build <app>` — and that works only for libs the app actually reaches. It does see lazily-loaded libs through `loadChildren`, so a feature wired up per step 4 is covered; a `ui` lib with no consumer yet is not. Give new libs a consumer, or accept that their templates are unchecked until they have one.
- **The grep is only as good as the directories you pass it.** Point it at this workspace's actual project roots (`apps libs`, `packages`, whatever `nx show projects` implies). Aimed at a directory that does not exist, it prints an error to stderr and nothing to stdout, which reads exactly like success.

## Definition of done

- [ ] Each new piece lives in the library type chosen in step 2. The app shell only gained a route entry and possibly a nav link.
- [ ] New libs were created by generators with explicit `--name`, `--importPath`, and `--unitTestRunner` matching the workspace's existing test runner.
- [ ] New projects carry the same kind of tags as their neighbours. If the workspace already uses `scope:*` and `type:*`, match it; if it has no tag scheme, do not invent one here — place the code correctly and raise the tag question with the user.
- [ ] Generator placeholder components that serve no purpose have been deleted, and so have their exports.
- [ ] The feature is reachable through `loadChildren` or `loadComponent`, with no eager imports of feature code from the app.
- [ ] Each touched `index.ts` exports only the public surface described above.
- [ ] `lint` passes on every touched project with the `@nx/enforce-module-boundaries` rule active, and nothing was added to `allow`, `depConstraints`, or `eslint-disable` to make it pass. If the workspace's constraints are only `sourceTag: '*'`, say so rather than reporting that boundaries were enforced.
- [ ] Every touched lib type-checks, the app builds, and the deep-import grep returns nothing.

If a boundary rule blocks the design, change the design: move the shared piece down into `ui`, `data-access`, `util`, or `shared`. Do not loosen the rule unless the user explicitly asks for that.
