# Event sourcing 02 — CQRS tracking projections

## Business context

Replaying every fulfillment event for every customer query is wasteful. Customers need a fast current view and timeline, while operations need counts and stuck-shipment dashboards.

## Read models

- `fulfillment_current`: current status, carrier, tracking number, last update.
- `fulfillment_timeline`: ordered customer-safe events.
- `fulfillment_operations`: aging/stuck metrics and retry state.

Projection consumers read committed domain/integration events and upsert idempotently by event id/stream version. Record projection position so rebuilding or resuming is explicit.

## Consistency contract

The write acknowledgement may precede projection visibility. Expose this eventual-consistency behavior, and use operation/version tokens when a client needs to wait for a read model to catch up.

## Replay safety

Separate pure projection handlers from side-effect subscribers. Rebuilding a read model must not resend emails, reissue payment actions, or publish duplicate external events.

## Tests and operations

Prove idempotent replay, out-of-order rejection/buffering policy, rebuild into a new table, atomic cutover, lag metrics, and schema-version compatibility.

## Use, avoid, and performance notes

Use dedicated projections when read shape/scale differs materially from writes. Avoid CQRS for a CRUD table whose single model is clear. Projection lag, rebuild duration, storage duplication, and catch-up throughput are explicit service-level indicators.
