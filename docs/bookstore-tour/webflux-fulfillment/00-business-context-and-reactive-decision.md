# Fulfillment 00 — Business context and reactive decision

## Business context

An accepted order is not the end of the customer journey. Warehouse and carrier systems produce a timeline—picked, packed, collected, in transit, out for delivery, delivered or failed. Customers need the current state and live updates without polling every few seconds.

## Request

Build a separate fulfillment API that:

- Starts fulfillment for an accepted order.
- Accepts idempotent carrier/warehouse events.
- Returns current state and complete timeline.
- Streams new tracking events through SSE.
- Survives slow subscribers and cancellation.

## Endpoints

```text
POST /api/v1/fulfillments
POST /api/v1/fulfillments/{id}/events
POST /api/v1/carrier-events
GET  /api/v1/fulfillments/{id}
GET  /api/v1/fulfillments/{id}/timeline
GET  /api/v1/fulfillments/{id}/stream
```

## Why WebFlux

The value comes from long-lived streaming connections and a non-blocking data/client path, not from replacing MVC annotations. Use WebFlux, coroutines, and R2DBC together; a WebFlux controller calling blocking JDBC would require isolation and would not be end-to-end reactive.

## Operational constraints

Define expected concurrent subscribers, event rate, disconnect behavior, retention, heartbeat strategy, and per-customer authorization before implementation. Backpressure controls producer/consumer imbalance; it does not create infinite memory or downstream capacity.

## Exit criteria

The team can explain why fulfillment is a separate bounded context and why its workload—not fashion—justifies a reactive API.

## Use and avoid

Use this stack for many long-lived streams with non-blocking persistence and clients. Avoid it when traffic is ordinary request/response CRUD, libraries block, or the team cannot operate reactive diagnostics. MVC remains the simpler default.
