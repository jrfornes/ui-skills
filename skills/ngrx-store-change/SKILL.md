---
name: ngrx-store-change
description: Change NgRx state as a complete unit instead of a half-finished one — every action gets the effect, reducer case, and selector it implies, and components never touch store internals. Use when adding or changing NgRx actions, effects, reducers, selectors, facades, feature state, or entity collections in an Angular app that uses @ngrx/store, or when the user mentions dispatching, loading flags, store-driven fetching, createFeature, createEntityAdapter, or a store facade.
---

# ngrx-store-change

NgRx changes fail by being incomplete, not by being wrong: an action nobody reduces, a reducer case nobody selects, an effect with no cancel or error path, a component that subscribes to raw store state and does the deriving itself. Every one of those compiles and most of them pass review.

So treat a store change as a single unit of work that ends at the feature's public surface. Work the steps in order.

```
- [ ] 1. Read the workspace's existing state conventions
- [ ] 2. Classify each trigger, then write down the actions it implies
- [ ] 3. Actions: one group per source, request / success / failure
- [ ] 4. Effects: one job each, an explicit cancel strategy, an error path that survives
- [ ] 5. Reducer: pure, immutable, one status field instead of boolean soup
- [ ] 6. Selectors: every value a component reads has a named selector
- [ ] 7. Facade: only if the workspace already uses them
- [ ] 8. Tests at the seams, then check the public surface
```

## 1. Read the workspace's existing state conventions

NgRx has several generations of API and most codebases mix them. Match what is already there; do not introduce a second style in one feature.

| Look at | To learn |
| --- | --- |
| `package.json`, the `@ngrx/*` versions | Which APIs you may use (see the version table below) |
| An existing state folder, e.g. `libs/*/data-access/src/lib/+state/` | File naming, whether selectors are separate from the reducer, whether `createFeature` is used |
| `provideStore(...)` / `StoreModule.forRoot(...)` | Standalone vs NgModule registration, and which `runtimeChecks` are on |
| `provideState` / `StoreModule.forFeature` call sites | Whether feature state is registered globally, at a route, or lazily |
| Any `*.facade.ts` | Whether the layer exists at all, and what it exposes |
| `createEntityAdapter` usage | The collection state shape to copy |
| `@ngrx/signals` in `package.json` | Whether new state is expected to be a SignalStore instead |

```bash
rg -n '"@ngrx/' package.json
rg -l 'createFeature|createActionGroup|createEntityAdapter|createEffect' --glob '*.ts'
rg -n 'provideStore|StoreModule.forRoot|runtimeChecks' --glob '*.ts'
rg -l 'Facade' --glob '*.ts'
```

| API | Available from |
| --- | --- |
| `createActionGroup`, `emptyProps` | `@ngrx/store` 14.3 |
| `createFeature` with `extraSelectors` | `@ngrx/store` 15 |
| `createEffect(..., { functional: true })` | `@ngrx/effects` 15 |
| `store.selectSignal` | `@ngrx/store` 16 |
| `concatLatestFrom`, `mapResponse`, `tapResponse` from `@ngrx/operators` | 17.2 (previously `concatLatestFrom` came from `@ngrx/effects`) |

**Check first that this belongs in the global store at all.** If exactly one component tree reads the state, nothing else observes it, and it dies with the route, a signal or a component-provided `signalStore` is the better home. The global store earns its cost when state is shared across features, survives navigation, or needs to be inspected in DevTools.

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

## 3. Actions: one group per source, request / success / failure

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
| `exhaustMap` | Submits that must not double-fire: save, login, "load more" | Ignores the new trigger until the current one finishes |
| `concatMap` | Writes whose order matters against each other | Queues, runs one at a time |
| `mergeMap` | Independent per-id work, e.g. deleting several rows | Runs in parallel, completion order not guaranteed |

**`catchError` goes inside the inner observable.** Placed on the outer stream it kills the effect: the error completes the `actions$` subscription and the effect silently stops handling every future action.

```ts
// Wrong — this effect works once, then dies forever.
actions$.pipe(
  ofType(OrdersPageActions.opened),
  switchMap(() => api.getOrders()),
  map((orders) => OrdersApiActions.ordersLoadedSuccess({ orders })),
  catchError(() => of(OrdersApiActions.ordersLoadedFailure({ message: 'Failed' }))),
);
```

If `@ngrx/operators` is available, `mapResponse` makes that mistake harder to write:

```ts
switchMap(() =>
  api.getOrders().pipe(
    mapResponse({
      next: (orders) => OrdersApiActions.ordersLoadedSuccess({ orders }),
      error: (error: unknown) =>
        OrdersApiActions.ordersLoadedFailure({ message: toMessage(error) }),
    }),
  ),
);
```

Also:

- Need state inside an effect? Use `concatLatestFrom(() => store.select(...))`, which subscribes only when the action arrives. `withLatestFrom` subscribes eagerly and can read state from before the action. Never `take(1)` a selector to grab a snapshot.
- **No effect-to-effect chains.** If an action's only purpose is to make a second effect run, and no reducer handles it, the two effects are one effect. Chains are only justified when the intermediate action is itself meaningful state that something reduces or that another feature listens to.
- Never dispatch an action the same effect listens to — that is an infinite loop.
- Side-effect-only effects (navigate, toast, analytics, `localStorage`) use `{ dispatch: false }`. Leaving `dispatch` on with a `tap` re-dispatches the source action forever.
- Return the observable. Never `.subscribe()` inside an effect, and never nest subscribes.
- Register the effect where its state is registered: `provideEffects(ordersEffects)` next to `provideState(ordersFeature)`, at the route for a lazy feature.

## 5. Reducer: pure, immutable, one status field instead of boolean soup

The reducer is a pure function of `(state, action)`. No `Date.now()`, no `Math.random()`, no HTTP, no router, no service calls, no logging. Anything non-deterministic belongs in the action payload, computed by the effect.

```ts
export interface OrdersState {
  orders: Order[];
  statusFilter: OrderStatus | null;
  status: 'idle' | 'pending' | 'success' | 'failure';
  error: string | null;
}

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
- **Do not store derived state.** No `filteredOrders`, no `orderCount`, no `selectedOrder` object. Store the inputs (`orders`, `statusFilter`, `selectedId`) and derive the rest in selectors, or the copies will go stale.
- Turn on the runtime checks so immutability violations fail loudly in development:

```ts
provideStore(
  {},
  {
    runtimeChecks: {
      strictStateImmutability: true,
      strictActionImmutability: true,
      strictStateSerializability: true,
      strictActionSerializability: true,
      strictActionTypeUniqueness: true,
    },
  },
);
```

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
- Parameterized reads: prefer putting the parameter in the store (the route param via `@ngrx/router-store`, whose selectors you create with `getRouterSelectors()`) and composing a plain selector. A selector *factory* called from a template or from several components at once destroys memoization, because `createSelector` memoizes only the most recent arguments.
- If a template needs more than about three values, expose one view-model selector and read it once, or use `store.selectSignal`.
- Export from the feature's public API (`index.ts`): the action groups, the selectors, the state type if consumers genuinely need it, and the `provideState` / `provideEffects` helpers. Do not export the reducer internals, the raw `initialState`, or effect functions.

## 7. Facade: only if the workspace already uses them

Check step 1. If the codebase has no facades, do not add the first one — inject `Store` in the container component and keep presentational children on `input()` / `output()`. If facades are the convention, the facade must actually hide the store:

```ts
@Injectable()
export class OrdersFacade {
  private readonly store = inject(Store);

  readonly orders = this.store.selectSignal(ordersFeature.selectVisibleOrders);
  readonly isLoading = this.store.selectSignal(ordersFeature.selectIsLoading);

  open(): void {
    this.store.dispatch(OrdersPageActions.opened());
  }

  filterByStatus(status: OrderStatus | null): void {
    this.store.dispatch(OrdersPageActions.statusFilterChanged({ status }));
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

Then read the diff against [references/anti-patterns.md](references/anti-patterns.md) and confirm the public surface is consistent: the actions, selectors, and facade methods a consumer sees should describe the feature, with no store internals among them.

## Definition of done

- [ ] Every trigger was classified as UI event, API result, or cross-feature, and its actions are named after events, grouped by source, with no action shared between sources.
- [ ] Every new action has the effect and reducer case its row in step 2 requires; every reducer case has a selector that exposes what it changed.
- [ ] Every request action has success and failure counterparts, and failure payloads are serializable.
- [ ] Each effect does one job, its flattening operator was chosen for its cancellation behaviour, and `catchError` (or `mapResponse`) sits inside the inner observable.
- [ ] No effect chains only to trigger another effect; no effect dispatches an action it listens for; side-effect-only effects are `{ dispatch: false }`.
- [ ] The reducer is pure and deterministic, updates immutably, stores no derived values, and the store's runtime checks are enabled.
- [ ] Components read state only through named selectors, and derived values are computed in selectors rather than in components or templates.
- [ ] A facade was added only because the workspace uses facades, and it exposes no `Store`, `dispatch`, actions, or selectors.
- [ ] Reducer, selector, and effect tests exist, including the failure path, and the existing suite plus lint still pass.
