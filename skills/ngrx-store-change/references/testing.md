# Testing seams for NgRx

Each layer has a seam that needs no rendered component. Use it.

Find the runner before writing a spec, and match it. In an Nx workspace the runner is per project, not a root `package.json` script: check the project's `test` target and look at a neighbouring spec. Recent Nx generators default Angular libraries to Vitest, while older projects in the same repo are usually on Jest, so one workspace can legitimately have both. Examples below use `jest.fn()`; `vi.fn()` is a drop-in, and on Jasmine the equivalents are `jasmine.createSpyObj` and `spy.and.returnValue`.

Cover, for every store change:

1. Reducer: request, success, and failure cases.
2. Selectors: each derived selector's projector.
3. Effect: success, failure, and the cancellation its operator promises.
4. Component or facade: that it reads through selectors and dispatches the right action.

## Reducers

A reducer is a pure function, so there is no TestBed, no async, and no mocking. `createReducer` returns its `initialState` for `undefined` state, which means you do not need the initial state exported to get at it:

```ts
import { ordersFeature } from './orders.reducer';

const initialState = ordersFeature.reducer(undefined, { type: '@@init' });
```

```ts
describe('ordersFeature.reducer', () => {
  it('marks the slice pending and clears a stale error when the page opens', () => {
    const state = ordersFeature.reducer(
      { ...initialState, error: 'Previous failure' },
      OrdersPageActions.opened(),
    );

    expect(state.status).toBe('pending');
    expect(state.error).toBeNull();
  });

  it('stores the loaded orders', () => {
    const state = ordersFeature.reducer(
      { ...initialState, status: 'pending' },
      OrdersApiActions.ordersLoadedSuccess({ orders: [orderA] }),
    );

    expect(state.orders).toEqual([orderA]);
    expect(state.status).toBe('success');
  });

  it('records the failure message', () => {
    const state = ordersFeature.reducer(
      { ...initialState, status: 'pending' },
      OrdersApiActions.ordersLoadedFailure({ message: 'Network error' }),
    );

    expect(state.status).toBe('failure');
    expect(state.error).toBe('Network error');
  });

  it('does not mutate the state it is given', () => {
    const before = { ...initialState, orders: [orderA] };
    ordersFeature.reducer(before, OrdersApiActions.ordersLoadedSuccess({ orders: [orderB] }));

    expect(before.orders).toEqual([orderA]);
  });
});
```

Assert on what the selectors expose rather than on internal field layout where you can — see the entity-adapter reference for collections.

## Selectors

Test the projector, not the selector. The projector is the pure function you wrote; calling the selector means constructing a whole root state, which couples the test to every slice's shape.

```ts
it('applies the status filter', () => {
  const visible = ordersFeature.selectVisibleOrders.projector(
    [pendingOrder, shippedOrder],
    'shipped',
  );

  expect(visible).toEqual([shippedOrder]);
});

it('returns everything when no filter is set', () => {
  expect(
    ordersFeature.selectVisibleOrders.projector([pendingOrder, shippedOrder], null),
  ).toEqual([pendingOrder, shippedOrder]);
});
```

The projector's arguments are the selector's inputs in order, which is also a design check: a projector taking five arguments usually means the selector should be composed from smaller named selectors.

For a selector composed of other selectors (`selectAll` from an entity adapter, for instance), the projector takes those inner results, not the state slice. When that gets awkward, call the selector against a minimal root state object instead:

```ts
expect(selectRoutedOrder({ orders: ordersState, router: routerState })).toEqual(orderA);
```

## Effects

The seam is `provideMockActions`: you push actions in and assert on the actions that come out, with the API stubbed.

### Class-based effects

```ts
describe('OrdersEffects', () => {
  let actions$: Observable<Action>;
  let effects: OrdersEffects;
  let api: { getOrders: jest.Mock };

  beforeEach(() => {
    api = { getOrders: jest.fn() };

    TestBed.configureTestingModule({
      providers: [
        OrdersEffects,
        provideMockActions(() => actions$),
        { provide: OrdersApi, useValue: api },
      ],
    });

    effects = TestBed.inject(OrdersEffects);
  });

  it('emits success with the loaded orders', (done) => {
    api.getOrders.mockReturnValue(of([orderA]));
    actions$ = of(OrdersPageActions.opened());

    effects.loadOrders$.subscribe((action) => {
      expect(action).toEqual(OrdersApiActions.ordersLoadedSuccess({ orders: [orderA] }));
      done();
    });
  });

  it('emits failure and keeps handling later actions', () => {
    api.getOrders.mockReturnValue(throwError(() => new Error('boom')));
    actions$ = of(OrdersPageActions.opened(), OrdersPageActions.refreshed());

    const emitted: Action[] = [];
    effects.loadOrders$.subscribe((action) => emitted.push(action));

    expect(emitted.length).toBe(2);
  });
});
```

That second test is the one that catches a misplaced `catchError`. Subscribing to the effect directly bypasses `defaultEffectsErrorHandler`, so the broken version emits its single failure action and completes, and every later action is ignored — assert on two emissions and it fails. A test that checks one emission passes against both versions. At runtime the error handler would resubscribe and hide this, which is why the seam is worth testing here rather than through the store.

### Functional effects

A functional effect is a function that calls `inject()`, so it has to be created inside an injection context:

```ts
TestBed.configureTestingModule({
  providers: [
    provideMockActions(() => actions$),
    { provide: OrdersApi, useValue: api },
  ],
});

const effect$ = TestBed.runInInjectionContext(() => loadOrders());
```

Everything after that is identical to the class-based case. If the effect also reads state, provide it: either `provideMockStore({ selectors: [{ selector: ordersFeature.selectOrders, value: [orderA] }] })` or the real store via `provideStore({})` and `provideState(ordersFeature)`.

### Cancellation

The operator choice is a behavioural promise, so test it with `TestScheduler` marbles. Inside `scheduler.run`, one marble character is one frame.

```ts
it('cancels an in-flight search when a newer term arrives', () => {
  const scheduler = new TestScheduler((actual, expected) => expect(actual).toEqual(expected));

  scheduler.run(({ hot, cold, expectObservable }) => {
    // 'ab' dispatched at frame 1, 'abc' at frame 3.
    actions$ = hot('-a-b', {
      a: OrdersPageActions.searchChanged({ term: 'ab' }),
      b: OrdersPageActions.searchChanged({ term: 'abc' }),
    });

    // Each request answers 3 frames after it is subscribed, then completes.
    api.search.mockReturnValue(cold('---r|', { r: [orderA] }));

    const effect$ = TestBed.runInInjectionContext(() => searchOrders());

    // Only the second request survives: subscribed at 3, emits at 6.
    expectObservable(effect$).toBe('------s', {
      s: OrdersApiActions.ordersLoadedSuccess({ orders: [orderA] }),
    });
  });
});
```

Swap `switchMap` for `concatMap` in the implementation and this test fails with two emissions, which is exactly what you want a write-path effect's test to assert instead.

### Side-effect-only effects

For `{ dispatch: false }` effects, assert on the collaborator and subscribe explicitly, since nothing will subscribe for you:

```ts
it('navigates to the new order', () => {
  const router = { navigate: jest.fn() };
  // ...provide router, then:
  actions$ = of(OrdersApiActions.orderCreatedSuccess({ id: '42' }));

  const effect$ = TestBed.runInInjectionContext(() => navigateToCreatedOrder());
  effect$.subscribe();

  expect(router.navigate).toHaveBeenCalledWith(['/orders', '42']);
});
```

## Components and facades

Use `provideMockStore` to isolate a component from state. Override the selectors it reads and spy on `dispatch`:

```ts
describe('OrdersPage', () => {
  let store: MockStore;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [OrdersPage],
      providers: [provideMockStore()],
    }).compileComponents();

    store = TestBed.inject(MockStore);
    store.overrideSelector(ordersFeature.selectVisibleOrders, [orderA]);
    store.overrideSelector(ordersFeature.selectIsLoading, false);
  });

  it('dispatches opened on init', () => {
    const dispatch = jest.spyOn(store, 'dispatch');
    TestBed.createComponent(OrdersPage).detectChanges();

    expect(dispatch).toHaveBeenCalledWith(OrdersPageActions.opened());
  });

  it('renders a row per visible order', () => {
    const fixture = TestBed.createComponent(OrdersPage);
    fixture.detectChanges();

    expect(fixture.nativeElement.querySelectorAll('[data-testid="order-row"]').length).toBe(1);
  });
});
```

Two things to know about `MockStore`:

- Changing an override mid-test requires `store.refreshState()`; overridden selectors do not re-emit on their own.
- Reset overrides with `store.resetSelectors()` in an `afterEach`, or an override leaks into the next spec file's expectations. It is an instance method, so inject the `MockStore` to call it.

`store.scannedActions$` is an alternative to spying on `dispatch` when you want to assert on a sequence.

The same approach tests a facade: provide `provideMockStore`, override the selectors it exposes, and assert that each intent method dispatches the right action. A facade test that has to reach into `Store` to set something up is telling you the facade is doing too little.

### Stories

If the workspace has Storybook, a container component that injects `Store` will not render in a story without providers, and the fix is the same `provideMockStore`:

```ts
const meta: Meta<OrdersPage> = {
  component: OrdersPage,
  decorators: [
    applicationConfig({
      providers: [provideMockStore({ selectors: [{ selector: ordersFeature.selectVisibleOrders, value: [orderA] }] })],
    }),
  ],
};
```

This is also a design signal: if a presentational component needs store providers to render a story, it is a container, and the state should be moving up to `input()` instead.

## Flow tests with the real store

`provideMockStore` never runs a reducer, so it cannot verify "dispatch this, state becomes that". When the thing under test is the wiring rather than one layer, use the real store with the feature registered and the HTTP boundary stubbed:

```ts
TestBed.configureTestingModule({
  providers: [
    provideStore({}),
    provideState(ordersFeature),
    provideEffects(ordersEffects),
    provideHttpClient(),
    provideHttpClientTesting(),
  ],
});

const store = TestBed.inject(Store);
store.dispatch(OrdersPageActions.opened());

TestBed.inject(HttpTestingController)
  .expectOne('/api/orders')
  .flush([orderA]);

expect(await firstValueFrom(store.select(ordersFeature.selectAllOrders))).toEqual([orderA]);
```

Keep a small number of these — one per feature covering the happy path and one failure — and let the per-layer tests carry the edge cases.
