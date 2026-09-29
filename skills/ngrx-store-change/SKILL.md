---
name: ngrx-store-change
description: Change NgRx state as a complete unit instead of a half-finished one — every action gets the effect, reducer case, and selector it implies, and components never touch store internals. Use when adding or changing NgRx actions, effects, reducers, selectors, facades, feature state, or entity collections in an Angular app that uses @ngrx/store, or when the user mentions dispatching, loading flags, store-driven fetching, createFeature, createEntityAdapter, or a store facade.
---

# ngrx-store-change

NgRx changes fail by being incomplete, not by being wrong: an action nobody reduces, a reducer case nobody selects, an effect with no cancel or error path, a component that subscribes to raw store state and does the deriving itself. Every one of those compiles and most of them pass review.

So treat a store change as a single unit of work that ends at the feature's public surface. Work the steps in order.

## 1. Read the workspace's existing state conventions

NgRx has several generations of API and most codebases mix them. Match what is already there; do not introduce a second style in one feature.

| Look at | To learn |
| --- | --- |
| `package.json`, the `@ngrx/*` versions | Which APIs you may use (see the version table below) |
| Whether `@ngrx/operators` is a dependency | Whether `concatLatestFrom`, `mapResponse`, and `tapResponse` are importable at all. They are not bundled with `@ngrx/effects` from v18 on |
| Whether `@ngrx/schematics` is a dependency | Whether generators exist for the files you are about to write (see step 3) |
| Whether `@ngrx/eslint-plugin` is in the ESLint config | Which of this skill's rules the lint run already enforces (see step 8) |
| An existing state folder, e.g. `libs/*/data-access/src/lib/+state/` | File naming, whether selectors are separate from the reducer, whether `createFeature` is used |
| `provideStore(...)` / `StoreModule.forRoot(...)` | Standalone vs NgModule registration, and which `runtimeChecks` were changed from their defaults |
| `provideState` / `StoreModule.forFeature` call sites | Whether feature state is registered globally, at a route, or lazily |
| Any `*.facade.ts` | Whether the layer exists at all, and what it exposes |
| `createEntityAdapter` usage | The collection state shape to copy |
| `@ngrx/signals` or `@ngrx/component-store` in `package.json` | Which local-state option this workspace actually has, for the check below |
| The unit test runner of the project you are touching | Whether specs use Jest, Vitest, or Jasmine. In an Nx workspace this is per project, not a root `package.json` script |

```bash
rg -n '"@ngrx/|"@nx/(jest|vitest)"' package.json
rg -n '@ngrx' eslint.config.* .eslintrc.json 2>/dev/null
rg -l 'createFeature|createActionGroup|createEntityAdapter|createEffect' --glob '*.ts'
rg -n 'provideStore|StoreModule.forRoot|runtimeChecks' --glob '*.ts'
rg -l 'Facade' --glob '*.ts'
npx nx show project <project> --json | rg -n '"test"' -A 5   # which runner this project uses
```

| API | Available from |
| --- | --- |
| `createActionGroup`, `emptyProps` | `@ngrx/store` 14.3 |
| `createFeature` | `@ngrx/store` 12.4, `extraSelectors` from 15 |
| `createEffect(..., { functional: true })` | `@ngrx/effects` 15 |
| `store.selectSignal` | `@ngrx/store` 16 |
| `concatLatestFrom` | `@ngrx/effects` up to 17.2, then `@ngrx/operators`. Removed from `@ngrx/effects` in 18 |
| `mapResponse`, `tapResponse` | `@ngrx/operators` 17.2. `tapResponse` came from `@ngrx/component-store`, which stopped re-exporting it |

Never write `concatLatestFrom` or `mapResponse` without confirming their source is installed; adding a dependency is the user's decision, and step 4 has fallbacks that always work.

**Check first that this belongs in the global store at all.** If exactly one component tree reads the state, nothing else observes it, and it dies with the route, the better home is a plain signal or a component-provided store — `signalStore` from `@ngrx/signals`, or `ComponentStore` with `provideComponentStore`, whichever the workspace has. The global store earns its cost when state is shared across features, survives navigation, or needs DevTools. This skill covers the global store only: if the answer is a SignalStore, say so and stop.

## 2. Classify each trigger, then write down the actions it implies

Split the request into triggers and label each one. The label decides who is allowed to dispatch it and what else you owe.

| Trigger | Action source | Dispatched by |
| --- | --- | --- |
| UI event — user clicked, typed, opened a page, changed a filter | `[Orders Page]` | The container component, or the facade it calls |
| API / backend result — response, error, websocket push | `[Orders API]` | Effects only. A component must never dispatch one |
| Cross-feature or lifecycle — route navigated, logged out, another feature's success action | Owned elsewhere | Nobody here; you only listen with `ofType` |

Then fill this in for every action before writing code. This table is the whole point of the skill:

| Action kind | Effect | Reducer case | Selector |
| --- | --- | --- | --- |
| UI event that needs server data | Yes — one effect that performs the request | Yes, if the UI shows progress (`status: 'pending'`, clear stale error) | Yes, for whatever the reducer changed |
| UI event that only changes view state (filter, sort, page, selection) | No | Yes | Yes — a derived selector that applies it |
| API success | Only for a follow-on side effect such as navigation or a toast, and that effect is `{ dispatch: false }` | Always | Yes — the data the UI renders |
| API failure | Only for logging or a toast, `{ dispatch: false }` | Always — `status: 'failure'` plus a serializable error | Yes, if the UI shows the error |
| Cross-feature action you listen to | Usually yes | Sometimes | — |

Two checks that catch most half-done work:

- If you cannot name the reducer case **or** the effect for an action you just added, the action is not needed.
- If you cannot name the selector that exposes state a reducer case just changed, that state is write-only. Add the selector or drop the state.

For a cross-feature trigger, prefer listening to the action that already exists (`routerNavigatedAction`, `AuthApiActions.loggedOut`) over asking the other feature to dispatch something new for you.

### Changing state that already exists

Adding to a slice is safe; changing one is not. An action's payload, a selector's return type, or a feature key can have consumers in libraries you are not looking at, so list them before editing.

```bash
rg -n 'OrdersApiActions\.|ordersFeature\.select' --glob '*.ts'  # dispatchers, ofType listeners, selector consumers
rg -n "'orders'" --glob '*.ts'                                  # the feature key, before renaming it
```

Then per consumer, either update it in the same change or keep the old action and selector alongside the new one. Renaming a feature key also breaks anything that persisted or rehydrated that slice, so check `metaReducers` first.

## 3. Actions: one group per source, request / success / failure

If `@ngrx/schematics` is installed, generate the file set rather than inventing file names. It is the only generator surface — the collections inside `@ngrx/store` and `@ngrx/effects` hold nothing but `ng-add` — and it can scaffold the whole set (`feature`, with `action`, `reducer`, `effect`, `selector`, and `entity` available individually).

```bash
npx nx g @ngrx/schematics:feature --help                                    # flags for THIS version
npx nx g @ngrx/schematics:feature orders --project=orders-data-access \
  --api --group --flat=false --dry-run
```

The templates are current: they emit `createActionGroup`, `emptyProps`, `props`, and a correctly nested `catchError`. Two things they do not do, and three edits the output needs:

- In a standalone workspace nothing is registered for you — `--module` only wires into an NgModule — so you still add the `provide*` function from step 6 yourself.
- The generated spec is a `should be created` smoke test. Replace it using step 8.
- Split the generated success and failure events into their own `[Orders API]` group; narrow the `props<{ error: unknown }>()` failure payload to a serializable message; and choose the effect style and flattening operator per step 4 rather than keeping the class-based `concatMap` default.

One `createActionGroup` per source, named after the event that happened — not after the mutation you want. `setLoading` and `updateOrdersArray` are reducer implementation details leaking into the event name; the next consumer will not fit.

```ts
// orders.actions.ts
export const OrdersPageActions = createActionGroup({
  source: 'Orders Page',
  events: {
    Opened: emptyProps(),
    Refreshed: emptyProps(),
    'Status Filter Changed': props<{ status: OrderStatus | null }>(),
  },
});

export const OrdersApiActions = createActionGroup({
  source: 'Orders API',
  events: {
    'Orders Loaded Success': props<{ orders: Order[] }>(),
    'Orders Loaded Failure': props<{ message: string }>(),
  },
});
```

Rules:

- **Never reuse one action across two sources.** If both the orders page and a nav button need a refresh, each gets its own action and the effect handles both with `ofType(OrdersPageActions.refreshed, OrdersToolbarActions.refreshClicked)`. Shared actions make DevTools useless for answering "what caused this?".
- Every request action gets a success and a failure counterpart. A request action with no failure counterpart means the failure path does not exist yet.
- Failure payloads carry a plain message or a small serializable error object, never an `HttpErrorResponse` — `strictActionSerializability` will reject it and it is not something a template can render anyway.
- Payloads stay minimal. Pass the id, not the whole entity the reducer already holds.

## 4. Effects: one job each, an explicit cancel strategy, an error path that survives

One effect does one job. If you are reaching for `if` inside an effect to decide which request to make, that is two effects.

```ts
export const loadOrders = createEffect(
  (actions$ = inject(Actions), api = inject(OrdersApi)) =>
    actions$.pipe(
      ofType(OrdersPageActions.opened, OrdersPageActions.refreshed),
      switchMap(() =>
        api.getOrders().pipe(
          map((orders) => OrdersApiActions.ordersLoadedSuccess({ orders })),
          catchError((error: unknown) =>
            of(OrdersApiActions.ordersLoadedFailure({ message: toMessage(error) })),
          ),
        ),
      ),
    ),
  { functional: true },
);
```

Pick the flattening operator deliberately; the default choice is the bug.

| Operator | Use for | What it does to an in-flight request |
| --- | --- | --- |
| `switchMap` | Reads where only the latest matters: search, typeahead, route param change, refresh | Cancels it. Never use for writes — the cancelled POST may still have hit the server |
| `exhaustMap` | Submits that must not double-fire: save, login, "load more" | Ignores the new trigger until the current one finishes. `mergeMap` here appends the same page twice on impatient clicks |
| `concatMap` | Writes whose order matters against each other | Queues, runs one at a time. On a typeahead every keystroke's request runs and the last one wins by accident |
| `mergeMap` | Independent per-id work, e.g. deleting several rows | Runs in parallel, completion order not guaranteed |

**`catchError` goes inside the inner observable**, as in the example above. On the outer stream it never produces the failure action the reducer is waiting for, so the slice sits in `pending` forever. The bug survives review because the framework resubscribes the effect and the next trigger appears to work; [references/anti-patterns.md](references/anti-patterns.md) explains what that costs. If `@ngrx/operators` is installed, `mapResponse({ next, error })` in place of `map` plus `catchError` makes the mistake harder to write, because there is no outer stream to put the handler on.

Also:

- Need state inside an effect? `concatLatestFrom(() => store.select(...))` subscribes to the selector only once the action arrives. `withLatestFrom` subscribes as soon as the effect does, which throws if the slice's feature is not registered yet; it does **not** read pre-action state, since reducers run before effects either way. Without `@ngrx/operators`, `concatMap((action) => of(action).pipe(withLatestFrom(store.select(...))))` is the same operator by hand. Never `take(1)` a selector to grab a snapshot.
- **No effect-to-effect chains.** If an action's only purpose is to make a second effect run, and no reducer handles it, the two effects are one effect. Chains are only justified when the intermediate action is itself meaningful state that something reduces or that another feature listens to.
- Never dispatch an action the same effect listens to — that is an infinite loop.
- Side-effect-only effects (navigate, toast, analytics, `localStorage`) use `{ dispatch: false }`. Leaving `dispatch` on with a `tap` re-dispatches the source action forever.
- Return the observable. Never `.subscribe()` inside an effect, and never nest subscribes.
- Register the effect where its state is registered, at the route for a lazy feature. `provideEffects` takes effect classes or a **record** of functional effects, so functional effects go in as a namespace object: `import * as ordersEffects from './orders.effects'`, then `provideEffects(ordersEffects)`. Passing one functional effect directly does not work. Step 6 wraps both calls in a single exported `provide*` function. Under module federation, see [references/anti-patterns.md](references/anti-patterns.md) for where to register.

## 5. Reducer: pure, immutable, one status field instead of boolean soup

The reducer is a pure function of `(state, action)`. No `Date.now()`, no `Math.random()`, no HTTP, no router, no service calls, no logging. Anything non-deterministic belongs in the action payload, computed by the effect.

```ts
export interface OrdersState {
  orders: Order[];
  statusFilter: OrderStatus | null;
  status: 'idle' | 'pending' | 'success' | 'failure';
  error: string | null;
}

const initialState: OrdersState = {
  orders: [],
  statusFilter: null,
  status: 'idle',
  error: null,
};

export const ordersFeature = createFeature({
  name: 'orders',
  reducer: createReducer(
    initialState,
    on(OrdersPageActions.opened, OrdersPageActions.refreshed, (state) => ({
      ...state,
      status: 'pending' as const,
      error: null,
    })),
    on(OrdersApiActions.ordersLoadedSuccess, (state, { orders }) => ({
      ...state,
      orders,
      status: 'success' as const,
    })),
    on(OrdersApiActions.ordersLoadedFailure, (state, { message }) => ({
      ...state,
      status: 'failure' as const,
      error: message,
    })),
  ),
});
```

- A single `status` union beats separate `loading` / `loaded` / `error` flags, which can represent states that cannot happen and always drift. Follow the repo's existing shape if it already picked one.
- Spread at every level you change, and use non-mutating array operations: `[...items, item]`, `items.filter(...)`, `items.map(...)`, `[...items].sort(...)`. Never `push`, `splice`, `sort`, or `reverse` on state. For collections, let `@ngrx/entity` do it — see [references/entity-adapter.md](references/entity-adapter.md).
- **Do not store anything you can derive from what you already store.** No `filteredOrders`, no `selectedOrder` object, no count of an array that is in state: those copies go stale. A value the server computed is not derived — a `totalCount` from a paginated response cannot be recovered from the page in hand, so it is an input and it belongs in state.
- **Decide what resets the slice.** State registered with `provideState` at a lazy route is not torn down when the route is left, so re-entering shows the last visit's data until something overwrites it. Reduce a reset on whichever trigger should clear it — `routerNavigatedAction`, `AuthApiActions.loggedOut`, or the page's own opened action.
- **Know which runtime checks you already have.** `strictStateImmutability` and `strictActionImmutability` are on by default in development, so a mutating reducer throws a `TypeError` today with no configuration, and every check is off in production builds. Only the two serializability checks and `strictActionTypeUniqueness` need opting into, and they have trade-offs — see [references/anti-patterns.md](references/anti-patterns.md).

## 6. Selectors: every value a component reads has a named selector

A component may only read state through an exported, named selector. No `store.select((state) => state.orders.items)`, no selecting a whole slice and picking fields in the template, no `combineLatest` in a component to join two slices.

`createFeature` already gives you `selectOrdersState` plus one selector per state property. Add derived ones next to it:

```ts
export const ordersFeature = createFeature({
  name: 'orders',
  reducer: /* ... */,
  extraSelectors: ({ selectOrders, selectStatusFilter, selectStatus }) => ({
    selectVisibleOrders: createSelector(selectOrders, selectStatusFilter, (orders, filter) =>
      filter ? orders.filter((order) => order.status === filter) : orders,
    ),
    selectIsLoading: createSelector(selectStatus, (status) => status === 'pending'),
  }),
});
```

- Derive in selectors, not components. Sorting, filtering, joining two slices, formatting a count — all selector work, all memoized, all testable without a TestBed.
- Parameterized reads: prefer putting the parameter in the store (the route param via `@ngrx/router-store`, whose selectors you create with `getRouterSelectors()`) and composing a plain selector. A selector *factory* called from a template creates a new selector on every change detection run, so nothing is ever memoized; one shared factory-made selector used with different arguments memoizes only the most recent call and thrashes. The older `props`-based selector form is deprecated in `@ngrx/store` and slated for removal, so do not reach for it either.
- If a template needs more than about three values, expose one view-model selector and read it once, or use `store.selectSignal`.
- Export from the feature's public API (`index.ts`) the action groups, the selectors, the state type if consumers genuinely need it, and one `provide*` function that registers everything. Keep the reducer, the raw `initialState`, and the effects themselves internal, so the effects record has no consumers outside the library:

```ts
// libs/orders/data-access/src/index.ts
export function provideOrdersState(): EnvironmentProviders {
  return makeEnvironmentProviders([provideState(ordersFeature), provideEffects(ordersEffects)]);
}
```

A lazy route then needs only `providers: [provideOrdersState()]`, which is also the `provide*()` surface the `nx-angular-feature` skill expects a data-access library to export.

## 7. Facade: only if the workspace already uses them

Check step 1. If the codebase has no facades, do not add the first one — inject `Store` in the container component and keep presentational children on `input()` / `output()`. A data-access library's public API is then its action groups, selectors, and `provide*` function rather than a facade class; both are legitimate contracts, and the workspace has already picked one. If facades are the convention, the facade must actually hide the store:

```ts
@Injectable()
export class OrdersFacade {
  private readonly store = inject(Store);

  readonly orders = this.store.selectSignal(ordersFeature.selectVisibleOrders);
  readonly isLoading = this.store.selectSignal(ordersFeature.selectIsLoading);

  open(): void {
    this.store.dispatch(OrdersPageActions.opened());
  }
}
```

A facade that leaks is worse than no facade. It must never expose `store`, a `dispatch(action)` passthrough, action creators, or selectors to its callers — those are all "inject the Store directly" wearing a hat. Its methods are named after user intent, and a component using it should not need to import anything from `@ngrx/*`.

## 8. Tests at the seams, then check the public surface

Each layer has a cheap seam; use it instead of driving everything through a rendered component. Details and runnable examples are in [references/testing.md](references/testing.md).

| Layer | How to test it |
| --- | --- |
| Reducer | Call `ordersFeature.reducer(state, action)` directly. No TestBed. Cover request, success, and failure |
| Selector | Call `selectVisibleOrders.projector(orders, filter)`. Never build a whole state tree |
| Effect | `provideMockActions` plus a stubbed API. Cover success, failure, and the cancellation the operator promises |
| Component / facade | `provideMockStore` with `overrideSelector`, assert on dispatched actions. Use the real store when the test is about a flow |

Write the specs in the runner that project already uses, which step 1 established. Do not introduce a second one.

Then run the checks, read the diff against [references/anti-patterns.md](references/anti-patterns.md), and confirm the public surface is consistent: the actions, selectors, and facade methods a consumer sees should describe the feature, with no store internals among them.

```bash
npx nx affected -t lint test                                  # or: nx run-many -t lint test -p <touched projects>
npx tsc -p libs/<scope>/data-access/tsconfig.lib.json --noEmit # consumers in other libs are not covered by one project's test run
rg -n 'store.select\(\(' --glob '*.ts'                         # inline selectors that should be named
rg -n 'take\(1\)|\.subscribe\(' --glob '*.effects.ts'          # snapshot reads and nested subscribes in effects
```

How much that lint run proves depends on whether `@ngrx/eslint-plugin` is configured: it enforces about a third of this skill mechanically, and without it the two greps are the only automated coverage. [references/anti-patterns.md](references/anti-patterns.md) maps its rules onto the sections above. Say which case you are in rather than implying lint checked any of this.

## Definition of done

- [ ] Every new action has the effect, reducer case, and selector its row in step 2 requires, and every request action has a serializable failure counterpart.
- [ ] Each effect's flattening operator was chosen for its cancellation behaviour, and `catchError` (or `mapResponse`) sits inside the inner observable.
- [ ] Every operator used is importable in this workspace: nothing from `@ngrx/operators` unless it is a dependency.
- [ ] The diff was read against steps 3–7 and [references/anti-patterns.md](references/anti-patterns.md).
- [ ] Something resets the slice when it should be empty, or the decision not to reset it was deliberate.
- [ ] Existing consumers of any action, selector, or feature key you changed were found and updated.
- [ ] Reducer, selector, and effect tests exist, including the failure path, written in the runner that project already uses, and `nx affected -t lint test` passes.
