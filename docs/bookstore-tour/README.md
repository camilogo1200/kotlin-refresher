# Bookstore curriculum map

The bookstore is a simple domain with intentionally professional depth. Each phase introduces complexity only after a business request creates a reason for it.

## System story

The first application owns the catalog and order intake. It begins as a small annotated CRUD API, becomes a feature-first hexagonal modular monolith, and proves the last-copy purchase race with a real database transaction. A fulfillment bounded context is then introduced as a separate reactive API because customers need live tracking streams. Finally, reliable events let fulfillment, notification, and analytics evolve independently.

```text
Bookstore API (MVC/JDBC)
  Catalog + Ordering
          |
          | OrderPlaced
          v
Fulfillment API (WebFlux/R2DBC)
          |
          | FulfillmentStatusChanged
          +----> Notification worker
          +----> Analytics worker
          +----> Customer SSE stream
```

## Learning order

### 1. Core modular monolith

1. [Business context](core/00-business-context.md)
2. [Kotlin domain foundations](core/01-kotlin-domain-foundations.md)
3. [Coroutines and Flow runway](core/02-coroutines-and-flow.md)
4. [First annotated CRUD slice](core/03-first-annotated-crud.md)
5. [DDD and a richer model](core/04-ddd-and-domain-model.md)
6. [Use cases, ports, and application services](core/05-use-cases-ports-and-services.md)
7. [Feature-first hexagonal refactor](core/06-hexagonal-refactor.md)
8. [Spring Data JDBC adapters](core/07-data-jdbc-adapters.md)
9. [HTTP contracts and Problem Details](core/08-http-contracts-and-errors.md)
10. [Catalog queries](core/09-catalog-queries.md)
11. [Orders, transactions, and concurrency](core/10-orders-transactions-and-locking.md)
12. [Outbound HTTP and resilience](core/11-outbound-http-and-resilience.md)
13. [Observability, testing, and release readiness](core/12-observability-testing-and-release.md)

### 2. Technology showroom

Use [the showroom](showroom/README.md) after the core API is stable. It compares adapters and runtimes; it does not invent `/books-jdbc` or `/books-r2dbc` endpoints.

### 3. Reactive companion API

Build the [fulfillment API](webflux-fulfillment/README.md). Its domain is separate, but it follows the same service-as-interactor and ports/adapters vocabulary.

### 4. Advanced distributed architecture

1. [Kafka and event-driven architecture](advanced/kafka-eda/README.md)
2. [Choreographed and orchestrated Sagas](advanced/sagas/README.md)
3. [Selective event sourcing and CQRS](advanced/event-sourcing-cqrs/README.md)

## Scope discipline

- The core API remains runnable without Kafka.
- The reactive API exists because its workload streams; WebFlux is not used for ordinary CRUD fashion.
- Microservices are split by bounded context and independent operational needs, never merely by framework.
- A Kafka log is not automatically an aggregate event store.
- Saga state is durable workflow state, not a long-running coroutine and not necessarily event sourcing.
- Notification is an eventual side effect; notification failure never rolls back a successful purchase.

## Every lesson must prove

- The business request and acceptance criteria are understood.
- Interface-to-implementation relationships are visible.
- Spring stereotypes match runtime roles.
- Blocking, concurrency, failure, and transaction boundaries are named.
- Tests demonstrate the most important behavior.
- Do/Don't guidance and industry trade-offs are recorded.

