# NgRx anti-patterns

Read the diff against this list before calling the change done. Ordered roughly by how much damage each one does.

Several of them are machine-checkable, so check for `@ngrx/eslint-plugin` before reviewing by hand:

| Rule | Enforced by |
| --- | --- |
| Components never subscribe to or derive from the store | `no-store-subscription`, `avoid-mapping-selectors`, `avoid-combining-selectors` |
| Every read goes through a named selector | `prefer-selector-in-select`, `prefix-selectors-with-select` |
| Actions are event-named and dispatched as creators | `good-action-hygiene`, `prefer-action-creator-in-dispatch` |
| No effect dispatches an action it listens for | `avoid-cyclic-effects`, `no-dispatch-in-effects` |
| `concatLatestFrom` over `withLatestFrom` | `prefer-concat-latest-from` |

`on-function-explicit-return-type` is worth knowing too: an explicit return type on each `on` handler is the alternative to the `as const` the reducer example uses to stop `status` widening to `string`.

| Anti-pattern | Symptom | Fix |
| --- | --- | --- |
| `catchError` on the outer effect stream | A failed request never produces its failure action, so the slice stays `pending` and the spinner never stops | Move it inside the flattening projection, or use `mapResponse` |
| Logic in the component | Component subscribes, computes, then dispatches based on what it read | Dispatch the event; decide in the effect or reducer; derive in a selector |
| Effect chained only to trigger another effect | An action with no reducer case and exactly one listener | Collapse into one effect |
| Store internals in the component | `store.select((s) => s.orders.items)`, or a slice picked apart in the template | Named exported selectors only |
| Derived state stored in the reducer | `filteredOrders`, `selectedOrder`, a count of an array already in state | Store the inputs, derive with `createSelector`. A server-computed `totalCount` is an input, not derived |
| Mutating state | `state.items.push(...)`, `items.sort()` | Non-mutating operations or the entity adapter. Already a `TypeError` in development: `strictStateImmutability` is on by default |
| One action reused by several sources | DevTools cannot tell you what caused a change | One action group per source; `ofType` both in the effect |
| Leaky facade | Facade returns `store`, or has `dispatch(action)` | Intent-named methods and exposed state only |
| Snapshot reads via `take(1)` | Effect or component pulls a value out of a selector imperatively | `concatLatestFrom` in the effect; bind in the template |
| `switchMap` on a write | Intermittent lost saves, orphaned server state | `concatMap` or `exhaustMap` |
| Global store for local state | A slice nothing outside one component tree ever reads | Signal, component service, or a component-provided `signalStore` |
| `{ dispatch: false }` forgotten | Infinite action loop as soon as the effect runs | Add it to side-effect-only effects |

## `catchError` in the wrong place

This is the most common NgRx bug, and the reason it survives review is not that it is invisible — it is that the framework papers over it.

When the error reaches the outer stream, the effect's subscription to `actions$` is torn down. `@ngrx/effects` wraps every effect in `defaultEffectsErrorHandler`, which passes the error to Angular's `ErrorHandler` and then resubscribes the effect, up to ten attempts before it stops trying. So the console does get an error, and the next trigger does work.

What is lost is everything that mattered:

- **The failure action is never dispatched.** The reducer never sees `ordersLoadedFailure`, so `status` stays `'pending'`, `error` stays `null`, and the UI shows a spinner forever. This is the symptom users report.
- **Stream state is discarded on resubscribe.** Anything the pipeline had accumulated — a `debounceTime` window, a `scan`, an in-flight request — is gone, so a resubscribed search or polling effect silently changes behaviour.
- **The tenth failure is the last.** After the retry budget is exhausted the effect really is gone for the rest of the session, so a backend having a bad minute can permanently disable a feature.
- **With `useEffectsErrorHandler: false` there is no budget at all** and the first error is permanent.

```ts
// Wrong: no failure action, and the pipeline is rebuilt behind your back.
loadOrders$ = createEffect(() =>
  this.actions$.pipe(
    ofType(OrdersPageActions.opened),
    switchMap(() => this.api.getOrders()),
    map((orders) => OrdersApiActions.ordersLoadedSuccess({ orders })),
    catchError(() => of(OrdersApiActions.ordersLoadedFailure({ message: 'Failed' }))),
  ),
);
```

```ts
// Right: the error is confined to the inner request.
loadOrders$ = createEffect(() =>
  this.actions$.pipe(
    ofType(OrdersPageActions.opened),
    switchMap(() =>
      this.api.getOrders().pipe(
        map((orders) => OrdersApiActions.ordersLoadedSuccess({ orders })),
        catchError((error: unknown) =>
          of(OrdersApiActions.ordersLoadedFailure({ message: toMessage(error) })),
        ),
      ),
    ),
  ),
);
```

`mapResponse` from `@ngrx/operators` is the same thing with the shape enforced, and `tapResponse` is its `{ dispatch: false }` sibling. Both need `@ngrx/operators` as a dependency; they are not part of `@ngrx/effects`, and `@ngrx/component-store` no longer re-exports `tapResponse`.

The test that catches it: dispatch the trigger, fail the API, then dispatch the trigger again and assert two actions came out. A single-emission test passes either way. That test subscribes to the effect observable directly, so there is no `defaultEffectsErrorHandler` in the way and the broken version really does emit once and complete — the unit test is stricter than the runtime, which is what you want here.

## Logic in the component

The component decided *what should happen*, which means the decision is untestable without rendering and unreachable from any other trigger.

```ts
// Wrong
onRefresh(): void {
  this.store.select(ordersFeature.selectOrders).pipe(take(1)).subscribe((orders) => {
    if (orders.length === 0) {
      this.store.dispatch(OrdersPageActions.opened());
    }
  });
}
```

```ts
// Right — the component reports the event.
onRefresh(): void {
  this.store.dispatch(OrdersPageActions.refreshed());
}
```

```ts
// Right — the condition lives in the effect, where state is available lazily.
// `concatLatestFrom` needs @ngrx/operators; see step 1 of the skill for the plain-RxJS form.
export const loadOrdersIfEmpty = createEffect(
  (actions$ = inject(Actions), store = inject(Store), api = inject(OrdersApi)) =>
    actions$.pipe(
      ofType(OrdersPageActions.refreshed),
      concatLatestFrom(() => store.select(ordersFeature.selectOrders)),
      filter(([, orders]) => orders.length === 0),
      switchMap(() => /* ... */),
    ),
  { functional: true },
);
```

The same rule covers the template: `*ngIf="(orders$ | async)?.length && !(loading$ | async)"` is a derivation. Give it a selector and a name.

## Effect-to-effect chains

```ts
// Wrong: `ordersFetchRequested` exists only so the second effect can hear it.
export const onPageOpened = createEffect(
  (actions$ = inject(Actions)) =>
    actions$.pipe(
      ofType(OrdersPageActions.opened),
      map(() => OrdersApiActions.ordersFetchRequested()),
    ),
  { functional: true },
);

export const fetchOrders = createEffect(
  (actions$ = inject(Actions), api = inject(OrdersApi)) =>
    actions$.pipe(ofType(OrdersApiActions.ordersFetchRequested), switchMap(() => /* ... */)),
  { functional: true },
);
```

Every hop adds a frame to DevTools, a place for the chain to be broken by an unrelated `ofType`, and a test. Collapse them: have `fetchOrders` listen to `OrdersPageActions.opened` directly.

An intermediate action is justified when it is genuinely part of the feature's vocabulary — something reduces it, another feature listens to it, or several triggers converge on it. "The next effect needs a trigger" is not that.

A stricter version of the same rule: an effect that both fetches and then navigates and then toasts is doing three jobs. Split by job, all listening to the same success action, rather than chaining them in sequence.

## Store soup in components

```ts
// Wrong
export class OrdersPage implements OnInit, OnDestroy {
  orders: Order[] = [];
  private sub = new Subscription();

  ngOnInit(): void {
    this.sub.add(
      this.store.select(ordersFeature.selectOrders).subscribe((orders) => {
        this.orders = orders.filter((o) => o.status === this.filter).sort(byDate);
      }),
    );
  }
}
```

Three problems in one: manual subscription management, deriving in TypeScript instead of a memoized selector, and `sort` mutating the array the store handed over — which already throws in development, because `strictStateImmutability` defaults to on.

```ts
// Right
export class OrdersPage {
  private readonly store = inject(Store);
  readonly orders = this.store.selectSignal(ordersFeature.selectVisibleOrders);
}
```

Manual `subscribe` in a component is defensible only when the reaction is imperative and not renderable (focusing an element, opening a dialog), and even then it usually belongs in an effect with `{ dispatch: false }`.

## Non-deterministic reducers

```ts
// Wrong
on(OrdersPageActions.noteAdded, (state, { text }) => ({
  ...state,
  notes: [...state.notes, { id: crypto.randomUUID(), text, at: Date.now() }],
})),
```

A reducer must produce the same output for the same input forever, or replaying the action log in DevTools produces a different app and every test needs its clock mocked. Generate the id and timestamp in the effect (or on the server) and put them in the action payload.

The same applies to HTTP calls, router navigation, `localStorage`, and logging inside a reducer or a selector. Selectors are also pure: no `new Date()` in a projector.

## Shared actions across sources

```ts
// Wrong: one action, three callers, no way to tell them apart.
export const loadOrders = createAction('[Orders] Load Orders');
```

Actions are events, and the event "the orders page was opened" is not the event "the refresh button was clicked" even when they currently do the same thing. Keep them separate and let the effect accept both; when one of them later needs different behaviour, nothing has to be untangled.

## Runtime checks

`strictStateImmutability` and `strictActionImmutability` are already on in development, and all checks are disabled in production builds. The three that need opting in are worth it, with one caveat each:

```ts
provideStore(
  {},
  {
    runtimeChecks: {
      strictStateSerializability: true,
      strictActionSerializability: true,
      strictActionTypeUniqueness: true,
    },
  },
);
```

The serializability pair bans `Date`, `Map`, `Set`, and class instances from state and action payloads — store timestamps as ISO strings and map DTOs to plain objects at the HTTP boundary. It also cannot be combined with `@ngrx/router-store`'s `FullRouterStateSerializer`, which stores a non-serializable router state; router-store logs a warning telling you so, and the default `MinimalRouterStateSerializer` has no such problem. `strictActionTypeUniqueness` throws at startup on duplicate type strings, which is what you want, but it will surface pre-existing duplicates the first time you enable it in an older codebase.

## Leaky facades

```ts
// Wrong: all three of these are just `Store` with extra steps.
export class OrdersFacade {
  readonly store = inject(Store);
  dispatch(action: Action): void { this.store.dispatch(action); }
  select<T>(selector: MemoizedSelector<object, T>): Observable<T> { return this.store.select(selector); }
}
```

If a component has to import an action creator or a selector to use the facade, the facade is not a boundary. Expose intent methods (`open()`, `filterByStatus(status)`, `retry()`) and read-only state (`orders`, `isLoading`, `error`).

Also: do not add a facade to a codebase that does not use them. The layer only pays off where it is applied consistently, and a lone facade next to twenty components that inject `Store` is just a third convention.

## Cancellation mistakes

| Situation | Wrong | Why it breaks |
| --- | --- | --- |
| Saving a form | `switchMap` | A double-click cancels the first request client-side; the server may have already committed it |
| Typeahead search | `concatMap` | Every keystroke's request runs; results arrive late and the last one wins by accident |
| Deleting several rows | `concatMap` when order does not matter | Serialises independent work for no reason |
| "Load more" button | `mergeMap` | Impatient clicking appends the same page twice |

## Testing anti-patterns

- Asserting on `ids` / `entities` instead of the exported selectors, so the test breaks when the slice's shape changes.
- Using `provideMockStore` to test a flow. It does not run reducers, so "dispatch then expect new state" can never pass. Use the real store with `provideState` for flow tests, and `provideMockStore` only to isolate a component from state.
- Testing effects by subscribing to the real `Store` and asserting on side effects two layers away, instead of feeding `provideMockActions` and asserting on the output action.
- Only testing the success path. The failure path is the half that is usually missing from the implementation.
