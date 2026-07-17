# Fulfillment 05 — Reliability, testing, and performance

## Business context

Streaming code can look correct with one client and fail under disconnect storms, slow collectors, database pressure, or event bursts. The team needs evidence rather than “reactive is faster” claims.

## Reliability model

- Use idempotency for inbound carrier events.
- Persist before broadcasting; reconnect reads durable history rather than relying on an in-memory hot stream alone.
- Define ordering per fulfillment id and reject/handle out-of-order status transitions.
- Make subscriptions cancellable and release resources in `finally`/completion hooks.
- Use timeouts around remote carriers, not around the entire indefinite SSE response.

## Test layers

- Domain transition tests.
- `runTest` application/Flow tests.
- R2DBC adapter tests with Testcontainers.
- `WebTestClient` contract and streaming tests.
- Load test with connected clients, publish rate, reconnect rate, pool size, and memory recorded.

## Performance questions

Measure event delivery latency, active subscribers, buffer depth, dropped/replayed events, database connections, event-loop utilization, heap, and cancellation cleanup. Compare only equivalent contracts and durability guarantees.

## Exit criteria

The service survives duplicate events, restarts, slow/disconnected clients, and measured load within stated limits. It is now ready to receive `OrderPlaced` through Kafka.

## Industry practice

Use a production-like load profile and preserve durability guarantees during testing. Avoid headline comparisons between MVC and WebFlux with different databases, payloads, or backpressure behavior; those benchmarks answer different questions.
