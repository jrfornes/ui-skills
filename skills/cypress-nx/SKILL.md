---
name: cypress-nx
description: Add or fix Cypress e2e coverage in an Nx monorepo so the spec lands in the e2e project that owns the app under test, reuses the page objects, commands, and fixtures already in the workspace, and actually runs under `nx affected -t e2e`. Use when a UI change needs end-to-end coverage, when an e2e spec is flaky or points at the wrong base URL, when `nx affected` skips the e2e project, or when the user mentions cypress.config.ts, e2e-ci, data-cy selectors, page objects, or Cypress component testing in an Nx workspace.
---

# cypress-nx

An e2e spec is only useful if CI runs it. In an Nx workspace that means the spec lives in the e2e project whose graph edge reaches the changed code, runs through an Nx target that owns the server and the base URL, and asserts on what a user sees rather than on how the app is built.

Work through the steps in order. Steps 1 and 2 are where the damage happens: a spec written at the wrong level or in the wrong project is worse than no spec, because it costs CI time on every run and still misses regressions.

## 1. Decide whether this belongs in e2e at all

Most "we need an e2e test" requests are really unit or component tests. Push the assertion to the lowest level that can still fail for the right reason.

| What changed | Test it with | Which project |
| --- | --- | --- |
| Pure function, pipe, validator, reducer, NgRx selector | Unit test | the `util` / `data-access` lib that owns it |
| Component rendering for given inputs, emitted outputs, conditional template branches | TestBed spec, or a Cypress component test where the workspace already configures one | the `ui` / `feature` lib that owns the component |
| Store wiring, effects, guards, resolvers, interceptors | Unit test with mocked HTTP | the `data-access` lib |
| A user path across routes: navigate, fill, submit, see the result | e2e | `<app>-e2e` |
| Auth, redirects, deep links, session restore, route guards end to end | e2e | `<app>-e2e` |
| Real browser behaviour: file upload or download, clipboard, multi-origin, print | e2e | `<app>-e2e` |

Rules of thumb:

- One e2e per critical path, not one per permutation. Validation messages, empty states, loading states, and error branches belong a level down, where they run in milliseconds and `nx affected` scopes them to the lib that changed instead of rebuilding and serving the whole app.
- Every spec you add becomes another `e2e-ci--<spec>` target that runs on every affected CI run. Adding coverage is not free.
- If the assertion can be written without a server, it does not need e2e.
- This is not licence to delete e2e coverage of a money path. Checkout, login, and publish flows earn their spec.

[references/test-level.md](references/test-level.md) has the cost of each level, worked examples of moving an assertion down a level, and how Cypress component testing is wired in Nx.

## 2. Find the e2e project that owns the change

Do not guess from folder names. Ask the project graph.

```bash
nx show projects --withTarget e2e            # every project that has an e2e target
nx show project <app>-e2e --json             # root, tags, targets, implicitDependencies
nx graph --focus=<changed-lib> --file=/tmp/graph.html
```

Then map the change:

1. Identify the project that holds the UI change (usually a `feature` or `ui` lib, sometimes the app shell).
2. Find which apps consume it. `nx graph --focus=<lib>` shows the consumers; `nx show projects --affected --type app` after touching the file confirms them.
3. The owning e2e project is the one whose `implicitDependencies` names that app, or whose `cypress.config.ts` web server command serves that app. Both are visible in `nx show project <app>-e2e --json`.

When a lib feeds two apps, the spec goes in the e2e project for the app where that path is a real user journey — not both. If the behaviour genuinely matters in both apps, it is component-level behaviour and belongs in a component test on the lib.

Never:

- create a `cypress/` folder at the workspace root, or an e2e project per feature;
- put a spec that calls `cy.visit()` inside a lib or inside the app's own `src/` — the e2e project's `specPattern` will not see it, and the component-testing target may try to run it;
- copy a spec into a second e2e project to "cover both apps".

[references/projects.md](references/projects.md) covers how Nx names and lays out e2e projects, the `cypress.config.ts` anatomy, inferred versus executor targets, and how the `affected` graph edge is actually formed.

## 3. Read that project's conventions before writing anything

The workspace's existing helpers beat anything you would invent. Read these first.

| Look at | To learn |
| --- | --- |
| `apps/<app>-e2e/cypress.config.ts` | base URL, which target serves the app, `specPattern`, viewport, retries, `env` |
| `src/support/commands.ts`, including its `declare namespace Cypress` block | custom commands that already exist (`cy.login`, seeding, selector helpers) |
| `src/support/*.po.ts` | the page-object style in use |
| `src/fixtures/` | fixture data already available |
| any existing `src/e2e/*.cy.ts` | selector convention, setup style, assertion style |
| `apps/<app>-e2e/tsconfig.json` `include` | whether a new folder will even be compiled |

Nx generates a function-per-query page object (`export const getGreeting = () => cy.get('h1')`) rather than a class. Follow whichever shape the workspace already uses. Extend `app.po.ts` or add a sibling `orders.po.ts` next to it; do not start a parallel `helpers/` or `pages/` tree.

A new custom command needs two edits, not one. Without the interface entry the spec fails to type-check. Keep the two `eslint-disable` comments Nx generates alongside the declaration — without them `@typescript-eslint/no-namespace` errors and `no-unused-vars` warns on `Subject`:

```ts
// apps/shop-e2e/src/support/commands.ts
// eslint-disable-next-line @typescript-eslint/no-namespace
declare namespace Cypress {
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  interface Chainable<Subject> {
    seedOrder(order: Partial<Order>): Chainable<string>;
  }
}

Cypress.Commands.add('seedOrder', (order) =>
  cy.request('POST', '/api/test/orders', order).its('body.id')
);
```

## 4. Write the spec

### Selectors

Find the convention before choosing one:

```bash
grep -rhoE 'data-(cy|test|testid|test-id)' --include='*.html' --include='*.ts' apps libs | sort | uniq -c
```

Use, in order: the attribute the workspace already uses, then `data-cy`, then an accessible role or name (`cy.contains`, or `cy.findByRole` where `@testing-library/cypress` is installed). If the element has no stable hook, add one to the template in the lib as part of the same change — that is a legitimate production edit, not test pollution.

Never select on: CSS classes, Angular Material internals (`.mat-mdc-form-field-infix`), Angular scoping attributes (`_ngcontent-*`), tag chains, `nth-child`, or generated ids. They change when someone restyles a component, and the failure will look like a broken feature.

### Waiting

Gate on the network and on the DOM, never on the clock.

```ts
describe('order list', () => {
  beforeEach(() => {
    cy.intercept('GET', '**/api/orders*').as('getOrders');
    cy.visit('/orders');
    cy.wait('@getOrders');
  });

  it('opens an order from the list', () => {
    cy.get('[data-cy=order-row]').first().click();
    cy.get('[data-cy=order-detail-title]').should('be.visible');
    cy.location('pathname').should('match', /^\/orders\/\w+$/);
  });
});
```

`cy.wait(500)` is always a bug: too short on a loaded CI machine, wasted time everywhere else. Cypress already retries assertions, so `should()` is the wait.

Assert on what the user sees. Reaching into the NgRx store, a component instance, or a class name means the test is a unit test wearing an e2e costume — and it will keep passing after the UI breaks.

### Data setup and teardown

Prefer, in this order:

1. `cy.intercept` with a fixture, when the test is about the UI path and not the backend contract.
2. `cy.request` against the app's test API to seed and to clean up.
3. `cy.task(...)` wired in `setupNodeEvents`, for database or CLI seeding.
4. Driving the UI, only when the setup steps are themselves under test.

Log in once per session rather than per test:

```ts
beforeEach(() => {
  cy.session('buyer', () => cy.request('POST', '/api/test/login', { user: 'buyer' }), {
    validate() {
      cy.request('/api/me').its('status').should('eq', 200);
    },
  });
});
```

`validate` must be a block body returning nothing. An arrow that implicitly returns the chainable fails to type-check, because Cypress expects `void | Promise<false | void>`. Without `validate` at all, an expired cached session is restored and the next test lands on a login screen.

Give each test its own data (a per-test suffix or generated id) so parallel CI shards do not collide, and clean it up in `afterEach` or by seeding fresh per spec. A spec must pass when run alone and when run twice in a row.

## 5. Run it through Nx targets

```bash
nx run <app>-e2e:e2e                                  # whole project; starts the app for you
nx run <app>-e2e:e2e --spec=src/e2e/orders.cy.ts      # one spec while iterating
nx run <app>-e2e:e2e --configuration=production       # against the built app, as CI does
nx open-cypress <app>-e2e                             # interactive, when the plugin defines it
nx affected -t e2e --base=origin/main                 # what CI will actually run
```

Confirm the target names in the `nx show project <app>-e2e --json` output from step 2 first — they are configurable, and older workspaces use the `@nx/cypress:cypress` executor instead of inferred targets.

Do not invent Cypress invocations around the target:

| Folklore | What to do instead |
| --- | --- |
| `cd apps/shop-e2e && npx cypress open` | `nx open-cypress shop-e2e`, or whatever `openTargetName` resolves to |
| `npx cypress run --config baseUrl=http://localhost:4200` | `nx run shop-e2e:e2e` — the target supplies the base URL and the server |
| `nx serve shop` in one terminal, Cypress in another | one command: the e2e target already has `dependsOn` on the serve target |
| Adding `cypress:*` scripts to the root `package.json` | Nx targets are the entry point; scripts bypass caching, the graph, and `affected` |
| Raising `defaultCommandTimeout` globally to fix one slow page | scope the timeout to the single assertion that needs it |
| Hardcoding `cy.visit('http://localhost:4200/orders')` | `cy.visit('/orders')` — the port differs between local, `serve-static`, and CI |

Two traps worth knowing before you trust a green run:

- The e2e target is cached. A repeat run can replay a cached pass without launching a browser. When you are chasing flake or verifying a fix, add `--skip-nx-cache`.
- The Nx Cypress preset reuses a server that is already answering on the base URL. A stale `nx serve` left running from an earlier branch will silently serve old code and your spec will test it. Stop stray dev servers before trusting a result; the log line to look for is `Reusing the server already running on ...`. Setting `webServerConfig: { reuseExistingServer: false }` in the preset options turns this into an error.

### Prove `affected` includes the project

A spec that CI never selects is not coverage.

```bash
git add -A
nx show projects --affected --withTarget e2e --base=origin/main
```

If the e2e project is missing from that list, the graph edge is broken — almost always a missing `implicitDependencies` entry on the e2e project. Fix the graph, not the CI command. [references/projects.md](references/projects.md) shows how the edge is formed and how to verify it.

Also check which target CI runs. Workspaces using the Nx atomizer run `nx affected -t e2e-ci`, which fans out into one task per spec file; a new spec is picked up automatically as long as it matches `specPattern`.

## Definition of done

- [ ] The new user path is covered by one spec in the e2e project that owns the app, inside the configured `specPattern`.
- [ ] The spec reuses existing page objects, custom commands, and fixtures, and new commands were added to the `Chainable` interface.
- [ ] Selectors, waits, and data setup follow step 4; the spec passes run alone and run twice in a row.
- [ ] `nx run <app>-e2e:e2e --spec=<new spec> --skip-nx-cache` is green, and so is the project's full `nx run <app>-e2e:e2e`.
- [ ] `nx show projects --affected --withTarget e2e` lists the e2e project for this change.
- [ ] `nx affected -t lint` passes, including the e2e project.

If the spec fails intermittently, fix the cause; do not paper over it with `retries`, longer timeouts, or `cy.wait`. [references/flake.md](references/flake.md) maps the common symptoms to their real causes.
