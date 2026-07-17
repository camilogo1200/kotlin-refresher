# Fulfillment 02 — R2DBC persistence

## Business context

The API should keep its database path non-blocking while serving many live connections. Fulfillment state and event history must remain durable and transactionally consistent within this service.

## Decision

Use Spring Data R2DBC coroutine repositories or `R2dbcEntityTemplate` behind output adapters. Flyway still runs migrations through JDBC at startup; runtime access uses R2DBC.

## Adapter structure

```text
FulfillmentRepository
  <- R2dbcFulfillmentRepository @Repository
      -> internal CoroutineCrudRepository/R2dbcEntityTemplate

FulfillmentEventRepository
  <- R2dbcFulfillmentEventRepository @Repository
```

Use `TransactionalOperator.executeAndAwait` for suspending transactional work and the Flow transactional extension only when the complete stream belongs in the transaction. Do not use imperative thread-bound transaction assumptions.

## Schema direction

Persist current fulfillment state plus an append-only tracking history. At this stage the event history is an audit/timeline, not necessarily the source of truth; event sourcing is introduced later.

## Tests and performance

Use Testcontainers with the actual R2DBC driver. Prove optimistic version conflicts, transaction rollback, ordered timeline queries, and cancellation. Connection pool size remains a hard limit even though waiting does not occupy servlet threads.

## Don't

- Do not wrap R2DBC operations in `withContext(Dispatchers.IO)`.
- Do not mix JDBC and R2DBC access inside one local transaction.
- Do not return Reactor types through application ports when Kotlin types suffice.

