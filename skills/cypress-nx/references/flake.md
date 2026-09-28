# Flaky traps and how to actually fix them

A flaky spec is a spec that lies. Once a team sees a red e2e job that passes on re-run, every red e2e job gets re-run, and the suite stops protecting anything. Fix the cause.

## Symptom to cause

| Symptom | Real cause | Fix |
| --- | --- | --- |
| Passes locally, fails in CI | CI is slower and the spec waits on a timer | Replace `cy.wait(<ms>)` with `cy.wait('@alias')` or a retried assertion |
| `cy.wait('@alias')` times out | The intercept pattern does not match the real request | Log the request in `cy.intercept(..., (req) => console.log(req.url))`, or widen to `'**/api/orders*'` |
| `element is detached from the DOM` | The element was re-rendered between the query and the action | Never store the element; re-query in one chain and let `should('be.visible')` settle first |
| Click lands on nothing, or hits the wrong element | An overlay, sticky header, or animation is still moving | Assert the overlay is gone first; disable animations rather than using `{ force: true }` |
| Assertion sees stale data | The store or an effect has not settled | Gate on the network alias that feeds it, then assert on the rendered DOM |
| First test passes, later ones fail | State leaked between tests, or tests depend on order | Seed in `beforeEach`; keep `testIsolation` on; make every test runnable alone |
| Fails only in a parallel CI shard | Two shards share seeded data | Unique per-test data, or a per-shard tenant |
| Fails around midnight or month end | Real clock or timezone | `cy.clock(new Date(...))`, and pin `TZ` for the e2e task |
| Passes, but suspiciously fast, with no browser output | Nx replayed a cached result | Re-run with `--skip-nx-cache` |
| Tests an old version of the app | A stale dev server was reused on the base URL | Kill stray servers; look for `Reusing the server already running on ...` in the log |

## Timing

The only correct waits are on the network and on the DOM. Cypress retries assertions for `defaultCommandTimeout`, so an assertion is a wait.

```ts
// wrong: guesses, and guesses differently on every machine
cy.visit('/orders');
cy.wait(1000);
cy.get('[data-cy=order-row]').should('have.length', 3);

// right: waits for the exact thing that has to happen
cy.intercept('GET', '**/api/orders*').as('getOrders');
cy.visit('/orders');
cy.wait('@getOrders');
cy.get('[data-cy=order-row]').should('have.length', 3);
```

Raise the timeout on a single slow step rather than globally:

```ts
cy.get('[data-cy=report-status]', { timeout: 30_000 }).should('contain', 'Ready');
```

`cy.wait('@alias')` on a request that is fired more than once resolves on the first. For a list that refetches after a mutation, alias each phase (`@getOrders`, `@createOrder`, `@getOrdersAfterCreate`) or assert on the resulting DOM instead.

## Animations

Angular Material and Angular's own animations are the most common source of detached-element and mis-targeted-click failures. Cypress waits for an element to stop moving (`waitForAnimations`, `animationDistanceThreshold`), but a dialog that fades while its content re-renders still slips through.

Best fix: serve the app under test with animations off. Gate it on Cypress so production is unaffected, matching whichever animation provider the app already uses:

```ts
// apps/shop/src/app/app.config.ts
const underTest = 'Cypress' in window;

providers: [
  // whichever the workspace already calls: provideAnimationsAsync, or
  // provideAnimations / provideNoopAnimations from @angular/platform-browser/animations
  provideAnimationsAsync(underTest ? 'noop' : 'animations'),
]
```

Where touching app config is not acceptable, disable CSS transitions from the support file:

```ts
// apps/shop-e2e/src/support/e2e.ts
beforeEach(() => {
  cy.document().then((doc) => {
    const style = doc.createElement('style');
    style.innerHTML = '*, *::before, *::after { transition: none !important; animation: none !important; }';
    doc.head.appendChild(style);
  });
});
```

Do not reach for `{ force: true }`. It skips the actionability checks that would have caught a genuinely unclickable button, so it converts a flaky test into a test that cannot fail.

## Angular change detection, NgRx, and signals

Cypress has no hook into Angular's change detection, and it does not need one: by the time the DOM has changed, the assertion retries will see it. Problems come from asserting on the wrong thing.

- Assert on rendered output, not on store state. Anything reached through `cy.window()` and a store reference is a unit test in disguise, and it passes even when the template is broken.
- After dispatching through the UI, wait on the effect's HTTP call, then on the DOM it produces.
- A spinner that appears and disappears faster than the test can see it is not worth asserting on; assert the final state instead. If the loading state matters, test it in a component test where you control the observable.
- `router.navigate` inside an effect resolves asynchronously. Assert with `cy.location('pathname').should(...)`, which retries, rather than reading the URL once.

## Test isolation and sessions

Cypress 12+ resets cookies, local storage, and the page between tests by default. Specs written for older versions that relied on state carrying across `it()` blocks break in ways that look random when specs are reordered or split across CI agents.

Under the Nx atomizer each spec file runs as its own task, possibly on a different machine, so cross-spec state is guaranteed not to survive. Every spec must set up everything it needs.

`cy.session` is the right way to avoid re-logging in: it caches and restores the session rather than skipping isolation.

```ts
const loginAs = (user: string) =>
  cy.session(user, () => cy.request('POST', '/api/test/login', { user }), {
    validate() {
      cy.request('/api/me').its('status').should('eq', 200);
    },
  });
```

Without `validate`, an expired cached session gets restored and the next test fails on a login screen. Write `validate` as a block body: an arrow that implicitly returns the chainable is a type error, since Cypress types the option as returning `void | Promise<false | void>`.

## Retries

`retries` in `cypress.config.ts` is a containment measure, not a fix:

```ts
retries: { runMode: 2, openMode: 0 }
```

It is defensible for genuine infrastructure noise. It is not defensible as a response to a spec you just wrote failing intermittently — a spec that needs a retry to pass will eventually need three. If a spec is retried in CI, treat it as a bug to fix, not a cost of doing business.

## Debugging a failure

```bash
nx run shop-e2e:e2e --spec=src/e2e/orders.cy.ts --skip-nx-cache
nx run shop-e2e:e2e --spec=src/e2e/orders.cy.ts --headed --no-exit   # watch it happen
nx open-cypress shop-e2e                                             # interactive runner
nx run shop-e2e:e2e --configuration=production --skip-nx-cache       # reproduce CI's build
```

Artifacts land in `dist/cypress/apps/shop-e2e/{videos,screenshots}` (or `apps/shop-e2e/test-output/cypress/...` in TS-solution setups). The atomized `e2e-ci--<spec>` targets write to a per-spec subfolder, which is where CI's uploaded artifacts come from.

For a CI-only failure, reproduce the CI conditions rather than guessing: the production configuration serves a built bundle through `serve-static` instead of the dev server, so missing assets, different base hrefs, strict production budgets, and tree-shaken dev-only code all show up only there. Also match the viewport — `viewportWidth`/`viewportHeight` from the config, not your monitor — because a responsive layout can hide the element CI is clicking.
