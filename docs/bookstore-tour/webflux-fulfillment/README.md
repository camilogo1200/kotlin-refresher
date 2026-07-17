# WebFlux fulfillment API

The fulfillment API is a separate bookstore bounded context and runnable application. It owns the evolving shipment timeline after an order is accepted.

## Why this API is reactive

Customers may keep tracking connections open, carrier events arrive asynchronously, and many subscribers observe long-lived streams. That workload provides a real reason for WebFlux, R2DBC, cancellation, backpressure, and `Flow`/SSE. Ordinary catalog CRUD remains in MVC.

## Lessons

1. [Business context and reactive decision](00-business-context-and-reactive-decision.md)
2. [Domain, use cases, and ports](01-domain-use-cases-and-ports.md)
3. [R2DBC persistence](02-r2dbc-persistence.md)
4. [WebFlux, Flow, and SSE](03-webflux-flow-and-sse.md)
5. [Carrier integration and webhooks](04-carrier-integration-and-webhooks.md)
6. [Reliability, testing, and performance](05-reliability-testing-and-performance.md)
7. [Functional routing variant](06-functional-routing-variant.md)

## Initial and later inbound adapters

Before Kafka, an internal HTTP endpoint starts fulfillment:

```text
StartFulfillmentController -> StartFulfillmentUseCase -> StartFulfillmentService
```

After Kafka, the same input port is reused:

```text
OrderPlacedKafkaConsumer -> StartFulfillmentUseCase -> StartFulfillmentService
```

Business logic does not move into the listener.

