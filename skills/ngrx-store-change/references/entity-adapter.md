# Entity adapter patterns

Before writing a new collection, copy the shape the repo already uses:

```bash
rg -n 'createEntityAdapter' --glob '*.ts' -A 6
rg -n 'getInitialState|getSelectors' --glob '*.ts'
```

If an existing feature already has a collection state, match its field names (`ids`/`entities` plus whatever it calls status and selection), its `selectId`, and whether it sorts in the adapter or in a selector. A second convention in the same codebase costs more than the one you dislike.

## When it applies

Use `@ngrx/entity` when the slice is **a keyed collection you look items up in**: orders, users, line items. You get O(1) lookup by id, generated CRUD helpers that are immutable by construction, and `selectAll` / `selectEntities` / `selectIds` / `selectTotal` for free.

Do not use it for:

- A single object (the current user, a settings blob, a wizard's form state).
- A short list that is only ever replaced wholesale and rendered in order — a plain array is less ceremony.
- A list whose items have no stable id. Synthesising one with the array index defeats the point, since every reorder rewrites every key.

## Setup

```ts
// orders.reducer.ts
export interface Order {
  id: string;
  reference: string;
  status: OrderStatus;
  placedAt: string;
}

export const ordersAdapter = createEntityAdapter<Order>({
  // Only needed when the key is not `id`.
  selectId: (order) => order.id,
  // Optional. When set, every insert keeps `ids` sorted; omit it to preserve insertion order.
  sortComparer: (a, b) => b.placedAt.localeCompare(a.placedAt),
});

export interface OrdersState extends EntityState<Order> {
  selectedId: string | null;
  status: 'idle' | 'pending' | 'success' | 'failure';
  error: string | null;
}

export const initialState: OrdersState = ordersAdapter.getInitialState({
  selectedId: null,
  status: 'idle',
  error: null,
});
```

`getInitialState` takes the extra, non-collection fields and merges them with `{ ids: [], entities: {} }`. Keep those extras flat next to the collection; do not nest the adapter state under another key, or you lose the free selectors.

`sortComparer` sorts the `ids` array, so `selectAll` comes out ordered. It does **not** apply to `selectEntities`. If the same collection needs two orders in different views, leave `sortComparer` off and sort in each selector instead — sorting twice in a selector is cheaper than keeping two collections.

## Collection operations

Every helper takes the update plus state and returns new state. They never mutate.

| Operation | Use when | Note |
| --- | --- | --- |
| `setAll(entities, state)` | A full list response replaces the collection | Drops anything not in the response |
| `setMany` / `setOne` | Upserting full replacements for specific items | `setOne` replaces the whole entity |
| `addMany` / `addOne` | Appending items known to be new | Ignores ids that already exist |
| `upsertMany` / `upsertOne` | Merging a partial page or a websocket push into what is there | Adds or shallow-merges |
| `updateMany` / `updateOne` | Patching known items | Takes `{ id, changes }`, not the entity |
| `mapOne` / `map` | Deriving new values from the current ones | `mapOne` takes `{ id, map: (e) => e }` |
| `removeMany` / `removeOne` / `removeAll` | Deleting | `removeMany` also accepts a predicate |

Two traps:

- `updateOne` takes an `Update<T>`, not an entity: `updateOne({ id: order.id, changes: { status: 'shipped' } }, state)`. Passing the entity is a compile error, because `changes` is missing — so this one fails loudly. Reach for `setOne` or `upsertOne` when you do have a whole entity in hand.
- Changing an entity's id via `changes` moves the key. That is legal and occasionally what you want after a server assigns a real id to an optimistically created row, but it will not preserve position unless a `sortComparer` re-sorts.

```ts
on(OrdersApiActions.ordersLoadedSuccess, (state, { orders }) =>
  ordersAdapter.setAll(orders, { ...state, status: 'success' }),
),
on(OrdersApiActions.orderUpdatedSuccess, (state, { order }) =>
  ordersAdapter.upsertOne(order, state),
),
on(OrdersApiActions.orderDeletedSuccess, (state, { id }) =>
  ordersAdapter.removeOne(id, state),
),
```

Note the shape: the extra fields are spread into the second argument, so one `on` handles both the collection change and the status change.

## Selectors

With `createFeature`, compose the adapter's selectors in `extraSelectors` so consumers never see `ids` or `entities`:

```ts
export const ordersFeature = createFeature({
  name: 'orders',
  reducer: createReducer(initialState /* ... */),
  extraSelectors: ({ selectOrdersState, selectSelectedId, selectStatus }) => {
    const { selectAll, selectEntities, selectTotal } =
      ordersAdapter.getSelectors(selectOrdersState);

    return {
      selectAllOrders: selectAll,
      selectOrderEntities: selectEntities,
      selectOrderCount: selectTotal,
      selectSelectedOrder: createSelector(
        selectEntities,
        selectSelectedId,
        (entities, id) => (id ? entities[id] ?? null : null),
      ),
      selectIsLoading: createSelector(selectStatus, (status) => status === 'pending'),
    };
  },
});
```

Without `createFeature`, the equivalent is `ordersAdapter.getSelectors(createFeatureSelector<OrdersState>('orders'))`.

Export only the composed selectors from the library's `index.ts`. `ids` and `entities` are storage details; a consumer that reads them will break the day the slice stops being an entity collection.

## Patterns

### The entity for the current route

Store the id, derive the entity. Never store the selected entity object — it goes stale the moment the collection updates.

```ts
// With @ngrx/router-store. The router selectors are created, not imported:
// `@ngrx/router-store` exports `getRouterSelectors`, not `selectRouteParams` itself.
export const { selectRouteParams } = getRouterSelectors();

export const selectOrderIdFromRoute = createSelector(
  selectRouteParams,
  (params) => params['orderId'] as string | undefined,
);

export const selectRoutedOrder = createSelector(
  ordersFeature.selectOrderEntities,
  selectOrderIdFromRoute,
  (entities, id) => (id ? entities[id] ?? null : null),
);
```

### Optimistic update with rollback

The effect needs the previous value to undo, so read it before dispatching the request and carry it on the failure action.

```ts
on(OrdersPageActions.orderStatusToggled, (state, { id, status }) =>
  ordersAdapter.updateOne({ id, changes: { status } }, state),
),
on(OrdersApiActions.orderStatusFailure, (state, { id, previousStatus, message }) =>
  ordersAdapter.updateOne({ id, changes: { status: previousStatus } }, { ...state, error: message }),
),
```

Use `concatMap` (not `switchMap`) for the effect: a cancelled write still reaches the server, and dropping the response means the rollback never happens.

### Per-item pending state

Do not add `loading: boolean` to the entity — it mixes server data with UI state, and the next `setAll` wipes it. Keep a set of ids alongside the collection:

```ts
export interface OrdersState extends EntityState<Order> {
  pendingIds: string[];
}
```

Then `selectIsPending` becomes a selector over `pendingIds`, and a list row asks for its own id.

### Server-side pagination

`selectTotal` counts the entities you are holding, which for a paginated collection is one page. The real total comes from the response, so store it — it is an input, not derived state — and keep it separate from the adapter's own count:

```ts
export interface OrdersState extends EntityState<Order> {
  totalCount: number;
  page: number;
}
```

Use `setAll` when a page replaces the list and `upsertMany` when pages accumulate, and never expose `selectTotal` as the collection's size to a paginator that needs `totalCount`.

### Nested data

If the response embeds children (`order.lineItems[]`) and those children are addressed or edited independently, give them their own adapter in their own slice and keep ids on the parent. Duplicating a child inside two parents means two places to update. If the children are only ever rendered with their parent and never edited alone, leave them embedded — normalising everything is its own anti-pattern.

## Testing

The adapter is already tested upstream; test your reducer through the adapter's own selectors rather than asserting on `ids` and `entities`, so the assertions survive a shape change.

`ordersAdapter.getSelectors()` with no argument returns selectors that read an `EntityState` directly, which is exactly what a reducer test has:

```ts
const { selectAll, selectTotal } = ordersAdapter.getSelectors();

it('replaces the collection on load success', () => {
  const state = ordersFeature.reducer(
    initialState,
    OrdersApiActions.ordersLoadedSuccess({ orders: [orderA, orderB] }),
  );

  expect(selectAll(state)).toEqual([orderB, orderA]); // sortComparer applies
  expect(selectTotal(state)).toBe(2);
  expect(state.status).toBe('success');
});
```

Note which form of `getSelectors` you are testing against. Called with no argument it returns plain functions of an `EntityState`, so `selectIds` and `selectEntities` have no `.projector` at all — call them directly on the reducer's output, as above. Called with a state selector, `getSelectors(selectOrdersState)` returns memoized selectors built as `createSelector(selectOrdersState, ...)`, so `selectAllOrders.projector(ordersState)` takes the slice. Either way you pass state, never `(ids, entities)`.
