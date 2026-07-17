# Fulfillment 03 — WebFlux, Flow, and Server-Sent Events

## Business context

Customers want new tracking events as they occur. Polling wastes requests and adds delay, while bidirectional WebSockets are unnecessary for a one-way server-to-client feed.

## HTTP contract

Return `text/event-stream` from:

```text
GET /api/v1/fulfillments/{id}/stream
```

The controller returns `Flow<FulfillmentEventResponse>` or a Flow of SSE wrappers. Use event ids so clients can reconnect with a last-event identifier when replay semantics are implemented. Send heartbeat comments if infrastructure closes idle connections.

## Flow responsibilities

- Repository/stream adapter emits domain events.
- Application query applies authorization-independent business filtering.
- HTTP adapter maps to the public event version.
- Cancellation from a disconnected client propagates to collection.

Choose buffering and overflow behavior intentionally. Dropping a tracking status can be unacceptable; a bounded buffer plus reconnect/replay is safer than unbounded memory.

## Error behavior

Once streaming response bytes are committed, the server cannot replace them with a normal Problem Detail. Validate identity and authorization before starting, encode each item safely, and treat mid-stream failures as termination plus observable logs/metrics.

## Tests

Use `WebTestClient` and virtual-time/Flow tests to prove ordered events, cancellation, heartbeat, reconnect, slow collector behavior, and no cross-customer leakage.

## Use, avoid, and production notes

Use SSE for one-way browser-friendly updates with automatic reconnect semantics. Avoid it for bidirectional collaboration or guaranteed offline delivery. Measure active connections, buffer depth, event latency, reconnect storms, and per-subscriber memory.
