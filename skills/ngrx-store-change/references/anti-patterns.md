# NgRx anti-patterns

Read the diff against the rules in SKILL.md steps 3–7 and the failure modes below before calling the change done.

Several rules are machine-checkable, so check for `@ngrx/eslint-plugin` before reviewing by hand:

| Rule | Enforced by |
| --- | --- |
| Components never subscribe to or derive from the store | `no-store-subscription`, `avoid-mapping-selectors`, `avoid-combining-selectors` |
| Every read goes through a named selector | `prefer-selector-in-select`, `prefix-selectors-with-select` |
| Actions are event-named and dispatched as creators | `good-action-hygiene`, `prefer-action-creator-in-dispatch` |
| No effect dispatches an action it listens for | `avoid-cyclic-effects`, `no-dispatch-in-effects` |
| `concatLatestFrom` over `withLatestFrom` | `prefer-concat-latest-from` |

`on-function-explicit-return-type` is worth knowing too: an explicit return type on each `on` handler is the alternative to the `as const` the reducer example uses to stop `status` widening to `string`.

## `catchError` in the wrong place — the spinner never stops

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

`mapResponse` enforces the right shape, and `tapResponse` is its `{ dispatch: false }` sibling; both need `@ngrx/operators`. The test that catches the wrong version is the two-emission effect test in [testing.md](testing.md).

## Logic in the component — subscribe, compute, then dispatch

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
// `concatLatestFrom` needs @ngrx/operators; see step 4 of the skill for the plain-RxJS form.
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

## Effect-to-effect chains — an action with no reducer case and one listener

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

## Store soup in components — manual subscriptions that derive and sort

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

Three problems in one: manual subscription management, deriving in TypeScript instead of a memoized selector, and `sort` mutating the array the store handed over, which throws in development.

```ts
// Right
export class OrdersPage {
  private readonly store = inject(Store);
  readonly orders = this.store.selectSignal(ordersFeature.selectVisibleOrders);
}
```

Manual `subscribe` in a component is defensible only when the reaction is imperative and not renderable (focusing an element, opening a dialog), and even then it usually belongs in an effect with `{ dispatch: false }`.

## Non-deterministic reducers — DevTools replay produces a different app

```ts
// Wrong
on(OrdersPageActions.noteAdded, (state, { text }) => ({
  ...state,
  notes: [...state.notes, { id: crypto.randomUUID(), text, at: Date.now() }],
})),
```

A reducer must produce the same output for the same input forever, or replaying the action log in DevTools produces a different app and every test needs its clock mocked. Generate the id and timestamp in the effect (or on the server) and put them in the action payload.

The same applies to HTTP calls, router navigation, `localStorage`, and logging inside a reducer or a selector. Selectors are also pure: no `new Date()` in a projector.

## Runtime checks

The three checks that need opting in are worth it, with one caveat each:

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

## Leaky facades — the facade exposes `store` or `dispatch(action)`

```ts
// Wrong: all three of these are just `Store` with extra steps.
export class OrdersFacade {
  readonly store = inject(Store);
  dispatch(action: Action): void { this.store.dispatch(action); }
  select<T>(selector: MemoizedSelector<object, T>): Observable<T> { return this.store.select(selector); }
}
```

If a component has to import an action creator or a selector to use the facade, the facade is not a boundary. Expose intent methods (`open()`, `filterByStatus(status)`, `retry()`) and read-only state (`orders`, `isLoading`, `error`).

## Module federation

Register a slice inside the remote that owns it and share `@ngrx/store` as a singleton. Unshared, host and remote get separate `Store` instances and dispatches never cross; shared, two remotes claiming one feature key overwrite each other.
