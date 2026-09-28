# Choosing the level: unit, component, or e2e

E2E tests are the most expensive and least precise tests in the workspace. They boot a browser, build and serve an app, and when they fail they tell you a page is wrong, not which unit is wrong. Use them where only they can work.

## Cost, per test, per CI run

| Level | Typical runtime | What `nx affected` scopes it to | What a failure tells you |
| --- | --- | --- | --- |
| Unit (`nx test <lib>`) | milliseconds | the lib that changed | the exact function |
| Component test (`nx component-test <lib>`) | a second or two | the lib that changed | the component and its template |
| E2E (`nx e2e <app>-e2e`) | tens of seconds, plus a build and a server | every app that consumes the changed lib, and their e2e projects | somewhere on this page |

The scoping column is the one people forget. A component test on `libs/orders/ui` runs when that lib changes. An e2e spec runs whenever anything the app depends on changes — so a badly placed e2e adds its cost to unrelated pull requests forever.

## Decision table

| Question | Answer | Level |
| --- | --- | --- |
| Is it a pure function, pipe, validator, reducer, or selector? | yes | unit |
| Is it "given these inputs, the template renders X" or "clicking this emits Y"? | yes | component |
| Is it store wiring, an effect, a guard, a resolver, or an interceptor? | yes | unit, with HTTP mocked |
| Does it need more than one route, or navigation between pages? | yes | e2e |
| Does it need a real session, redirect, or deep link? | yes | e2e |
| Does it need a real browser capability — upload, download, clipboard, another origin? | yes | e2e |
| Does it need the real backend contract, not a mock of it? | yes | e2e (or an API contract test) |
| Is it another variation of a path already covered by an e2e? | yes | push the variation down to component or unit |

When two answers conflict, take the lowest level that can still fail for the right reason.

## Moving an assertion down a level

**Form validation.** One e2e proves the happy path submits and the user lands on the confirmation page. The twelve invalid-input permutations belong in a component test or a TestBed spec on the form component, where each one runs in milliseconds and names the field that broke.

**Empty, loading, and error states.** These are template branches driven by inputs. Drive them directly in a component test instead of contorting an e2e with `cy.intercept` stubs that force a 500 and a network delay.

**Table sorting and filtering.** Component test on the table component with a fixed dataset. The e2e only needs to prove the table is reachable and populated with real data.

**Permission-dependent UI.** Component test per role, driven by an input or a stubbed service. Keep one e2e for the "logged-in user without permission is redirected" path, because that one crosses routes and involves a guard.

**Formatting and copy.** Unit test on the pipe. An e2e asserting exact wording will break on every copy edit and teach the team to ignore e2e failures.

## Cypress component testing in Nx

Component tests give you a real browser and real styles without a server, which fits Angular components with meaningful templates, CSS, or interactions. They are per-project, so they belong in the lib that owns the component.

Set up a project once:

```bash
nx g @nx/angular:cypress-component-configuration --project=orders-ui --dry-run
```

Useful flags: `--buildTarget=shop:build` to point at the app whose build configuration should compile the components, and `--generateTests` to scaffold a `.cy.ts` next to every existing component.

Scaffold a test for one component:

```bash
nx g @nx/angular:component-test --project=orders-ui \
  --componentDir=src/lib/order-card \
  --componentFileName=order-card \
  --componentName=OrderCard
```

The `@nx/cypress/plugin` entry in `nx.json` names the target, `component-test` by default:

```bash
nx run orders-ui:component-test
nx affected -t component-test
```

Component test specs are also `*.cy.ts`, but they live inside the lib next to the component, not in the e2e project. Keep the two apart: a spec that calls `cy.visit()` belongs in the e2e project, and a spec that calls `cy.mount()` belongs in the lib.

```ts
// libs/orders/ui/src/lib/order-card/order-card.cy.ts
import { createOutputSpy } from 'cypress/angular';
import { OrderCard } from './order-card';

describe(OrderCard.name, () => {
  it('emits select with the order id', () => {
    cy.mount(OrderCard, {
      componentProperties: {
        order: { id: 'o-1', total: 42 },
        select: createOutputSpy('selectSpy'),
      },
    });
    cy.get('[data-cy=order-card]').click();
    cy.get('@selectSpy').should('have.been.calledWith', 'o-1');
  });
});
```

`createOutputSpy` returns an `EventEmitter` with a spied `emit`, aliased under the name you pass. It works for both `@Output()` emitters and signal-based `output()`. `cy.mount(Component, { autoSpyOutputs: true })` spies on every output at once, aliased as `<outputName>Spy`.

If the workspace has no component-testing configuration and does not want one, use Angular's `TestBed` through the existing `nx test <lib>` setup instead. Do not introduce a third testing stack to avoid writing one e2e.

## What only e2e can do

Keep e2e for the things no lower level reproduces:

- The app actually boots — providers resolve, the router configuration is valid, lazy chunks load.
- Routing across features, including guards, redirects, and deep links into a lazy route.
- Real authentication and session restore across a reload.
- The frontend and backend agreeing on a contract, where the test runs against a real API.
- Browser-level behaviour: file upload and download, multi-origin flows, print, clipboard.

A workspace with a handful of these, all green and all fast, is worth more than a hundred e2e specs that nobody trusts.
