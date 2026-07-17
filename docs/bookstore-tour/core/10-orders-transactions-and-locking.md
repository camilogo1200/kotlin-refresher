# Core 10 — Orders, transactions, and the last-copy race

## Business context

Two customers attempt to buy the final copy. The system must produce one committed order and one understandable conflict, without negative stock, partial order lines, or a remote call holding database resources.

## Request and acceptance criteria

`POST /api/v1/orders` validates the request, obtains trusted price quotes, atomically decrements stock, computes totals, and persists the order aggregate.

- No database transaction is open during remote price calls.
- Order, items, and stock updates commit or roll back together.
- Two concurrent final-copy requests yield one success and one `409`.
- Retrying an HTTP request does not accidentally create duplicate orders once idempotency is introduced.

## Interaction flow

```text
OrderController
  -> PlaceOrderUseCase
      <- PlaceOrderService @Service (suspending orchestration)
          -> PriceProvider (remote calls, outside transaction)
          -> OrderCommitPort
              <- JdbcOrderCommitAdapter @Repository
                  -> imperative @Transactional delegate
```

The transactional delegate is a separate Spring bean so the call crosses the proxy. Its imperative method contains no suspension point: load/version-check books, invoke domain stock behavior, calculate total from verified quotes, and persist.

## Concurrency choice

Use Spring Data `@Version` optimistic locking for the low-contention last-copy race. Map `OptimisticLockingFailureException` to a conflict. Discuss pessimistic locking only for truly hot rows where retries become the common case.

## Transaction traps

- Self-invocation bypasses proxy advice.
- Default rollback follows exception type even though Kotlin has no checked-exception syntax.
- `readOnly = true` expresses intent but does not turn JDBC into a cache or remove every database cost.
- Do not wrap remote HTTP and database work in one long transaction.

## Tests and proof

Run a two-request integration test with latches/barriers so the race is repeatable. Assert final stock, one order, one conflict, and no partial rows. Also prove rollback after an order-line failure.

## Exit criteria

The monolith owns the invariant correctly with one local transaction. Distributed consistency is deferred until bounded contexts own separate data. Continue to [outbound HTTP and resilience](11-outbound-http-and-resilience.md).

## Use, avoid, performance, and production notes

Use one local transaction while one database owns the invariant; it is faster and easier to reason about than a Saga. Avoid retrying optimistic conflicts blindly when user-visible price/stock may have changed. Monitor transaction duration, pool saturation, deadlocks, and conflict rate before changing lock strategy.
