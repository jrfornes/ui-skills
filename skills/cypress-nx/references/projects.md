# E2E project layout, targets, and the affected graph

Verified against Nx 23.2.1 with `@nx/angular` and `@nx/cypress` 23.x. Older majors differ where noted. Always confirm with `nx show project <name> --json` rather than trusting a folder name.

## How the workspace names e2e projects

| Layout | Project name | Where it comes from |
| --- | --- | --- |
| `apps/shop-e2e/` | `shop-e2e` | Nx default: the app generator with `--e2eTestRunner=cypress` creates a sibling project |
| `apps/shop/shop-e2e/` | `shop-e2e` | some workspaces nest the e2e project under the app directory |
| `apps/<domain>/shop-e2e/` | `shop-e2e` | nested app directories keep the `-e2e` suffix on the last segment |
| `e2e/shop/` | varies | workspaces that group all e2e projects in one top-level folder |
| package-name project (`@org/shop-e2e`) | from `package.json` `name` | TS-solution setups, where the project name comes from `package.json`, not `project.json` |

Find them all rather than pattern-matching on a suffix:

```bash
nx show projects --withTarget e2e
find . -name 'cypress.config.*' -not -path '*/node_modules/*'
```

If a project has no `e2e` target but has a `cypress.config.ts`, the plugin's `targetName` has been renamed. Check the `@nx/cypress/plugin` entry in `nx.json`.

## Layout

The default is `src/e2e/*.cy.ts` for specs, with `src/support/` and `src/fixtures/` alongside. Older workspaces may have `src/integration/*.spec.ts` (Cypress ≤ 9) or a `cypress/` directory instead of `src/` when `cypressDir` is set to `cypress`. Put new specs wherever the existing ones are.

## `cypress.config.ts` anatomy

```ts
const { nxE2EPreset } = require('@nx/cypress/plugins/cypress-preset');
const { defineConfig } = require('cypress');

module.exports = defineConfig({
  e2e: {
    ...nxE2EPreset(__filename, {
      cypressDir: 'src',
      webServerCommands: {
        default: 'npx nx run shop:serve',
        production: 'npx nx run shop:serve-static',
      },
      ciWebServerCommand: 'npx nx run shop:serve-static',
      ciBaseUrl: 'http://localhost:4200',
    }),
    baseUrl: 'http://localhost:4200',
  },
});
```

What the preset fills in, derived from `cypressDir`:

| Setting | Value with `cypressDir: 'src'` |
| --- | --- |
| `specPattern` | `src/**/*.cy.{js,jsx,ts,tsx}` — anywhere under `src`, not only `src/e2e` |
| `supportFile` | `src/support/e2e.{js,ts}` |
| `fixturesFolder` | `src/fixtures` |
| `videosFolder` | `dist/cypress/apps/shop-e2e/videos` (`test-output/cypress/videos` in TS-solution setups) |
| `screenshotsFolder` | `dist/cypress/apps/shop-e2e/screenshots` (same exception) |
| `chromeWebSecurity` | `false` |

The preset also installs a `setupNodeEvents` hook that starts the web server. Its behaviour matters:

- If something already answers on `baseUrl`, it logs `Reusing the server already running on <url>` and does **not** start its own. A stale dev server from another branch will be tested instead of your code. Set `webServerConfig: { reuseExistingServer: false }` to make that an error instead.
- Otherwise it spawns the web server command and waits up to 60 seconds for the base URL to respond. Raise it with `webServerConfig: { timeout: ... }` if a cold production build needs longer.

If the project defines its own `setupNodeEvents` for `cy.task`, it must be composed with the preset's, not written over it, or the web server never starts.

## Where the base URL comes from

| Mode | Base URL | Server |
| --- | --- | --- |
| `nx run shop-e2e:e2e` | `baseUrl` in `cypress.config.ts` | `webServerCommands.default`, via the target's `dependsOn` on `shop:serve` |
| `nx run shop-e2e:e2e --configuration=production` | same `baseUrl` | `webServerCommands.production` |
| `nx run shop-e2e:e2e-ci` (atomizer) | `ciBaseUrl`, injected with `--config` | `ciWebServerCommand`, via `dependsOn` on `shop:serve-static` |
| Executor-based (`@nx/cypress:cypress`) | derived from `devServerTarget`, or the explicit `baseUrl` option | the `devServerTarget` |

Consequences for specs: always `cy.visit('/orders')`, never an absolute URL. Never read a port from an environment variable you set yourself. If a spec needs an external URL (an identity provider, for instance), put it in the config's `env` block and read it with `Cypress.env(...)`.

Running two e2e projects at once needs distinct ports. `nx g @nx/cypress:configuration` takes `--port`, where `0` asks the dev server for a free port and `cypress-auto` is the fallback for servers that cannot do that.

## Inferred targets versus executor targets

Modern workspaces register `@nx/cypress/plugin` in `nx.json`, whose options name the targets: `targetName` (`e2e`), `openTargetName` (`open-cypress`), `componentTestingTargetName` (`component-test`), and `ciTargetName` (`e2e-ci`). `project.json` then has an empty `targets` block and the plugin infers these from `cypress.config.ts`:

| Target | What it runs | `dependsOn` |
| --- | --- | --- |
| `e2e` | `cypress run` in the project directory; cached | `shop:serve` |
| `e2e-ci--src/e2e/app.cy.ts` | `cypress run --spec <that file>` with per-spec video and screenshot folders | `shop:serve-static` |
| `e2e-ci` | `nx:noop` that depends on every `e2e-ci--*` target | all of them |
| `open-cypress` | `cypress open` | none |

The per-spec `e2e-ci--*` targets appear and disappear automatically as spec files are added and removed — that is the "atomizer", and it is what lets CI distribute specs across agents. A new spec needs no configuration, only a path matching `specPattern`.

Older workspaces instead have an explicit target:

```json
"e2e": {
  "executor": "@nx/cypress:cypress",
  "options": {
    "cypressConfig": "apps/shop-e2e/cypress.config.ts",
    "devServerTarget": "shop:serve",
    "testingType": "e2e"
  },
  "configurations": {
    "production": { "devServerTarget": "shop:serve:production" }
  }
}
```

Here `--watch` opens the interactive GUI instead of running headlessly, and there is no `e2e-ci`. Other executor-only options worth knowing: `--skipServe` runs against an already-serving app, `--port` overrides the port passed to the `devServerTarget` (`cypress-auto` picks a free one), and `--headed` shows the browser. `nx g @nx/cypress:convert-to-inferred` migrates a project to the plugin; only run it if the user asked for that migration.

Either way `--spec` works, so `nx run shop-e2e:e2e --spec=src/e2e/orders.cy.ts` narrows the run in both. On the executor it is a comma-delimited glob string; on an inferred target it is passed straight through to `cypress run`.

## The affected graph edge

The e2e project has no source-level import of the app, so Nx cannot infer the dependency. It comes from one line in the e2e project's `project.json`: `"implicitDependencies": ["shop"]`.

That line is load-bearing. With it, editing `apps/shop/src/app/app.ts` marks both `shop` and `shop-e2e` affected. Without it, only `shop` is affected and `nx affected -t e2e` runs nothing — CI stays green while the flow is broken. Run the `affected` check from SKILL.md step 5 after any change to an e2e project, and whenever you create one.

Other ways the edge goes missing:

- The e2e project depends on an API or backend app that is not listed. Add it to `implicitDependencies` so backend changes also trigger the suite.
- A spec imports types from a lib (`@org/orders/data-access`). That creates a real graph edge and is fine, but it does not replace the edge to the app.
- `nx affected` compares against `defaultBase` in `nx.json`. If the branch was cut from somewhere else, pass `--base=<ref>` explicitly, or the affected set is wrong in both directions.

## Tags and boundaries

E2E projects are usually tagged `type:e2e` plus the app's scope. Where `@nx/enforce-module-boundaries` has real `depConstraints`, an untagged e2e project cannot import any library at all, and the spec fails lint with `A project without tags matching at least one constraint cannot depend on any libraries`.

```js
{ sourceTag: 'type:e2e', onlyDependOnLibsWithTags: ['type:util', 'type:data-access'] }
```

Keep e2e imports to types, constants, and test utilities. A spec importing a `type:ui` component is a sign the assertion belongs in a component test.

## Creating a new e2e project

Only when a new app has none. Prefer generating the app with e2e attached:

```bash
nx g @nx/angular:app apps/shop --e2eTestRunner=cypress
```

To add Cypress to an app that already exists:

```bash
nx g @nx/cypress:configuration --project=shop-e2e --devServerTarget=shop:serve --dry-run
```

Run with `--dry-run` first, then check `nx list @nx/cypress` for the generators the installed version actually has. Afterwards, replace the generated `app.cy.ts` smoke test with a real one rather than leaving the `Welcome` placeholder, and confirm `implicitDependencies` was set.
