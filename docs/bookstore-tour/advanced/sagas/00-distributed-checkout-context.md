# Saga 00 — Distributed checkout business context

## The business change

The original modular monolith could place an order, decrement stock, and persist its lines in one database transaction. The bookstore has grown:

- Inventory is managed independently across warehouse locations.
- A payment provider authorizes customer payment methods.
- Fulfillment owns shipment creation and carrier selection.
- Ordering still owns the customer-facing purchase decision.

The bookstore cannot hold one database transaction across these owners. It also cannot tell a customer “order confirmed” when stock was never reserved, payment was declined, or fulfillment cannot serve the address.

## New use case

`CheckoutOrderUseCase` checks out an existing `DRAFT` order.

This is an **alternative advanced evolution** of Core 10, not an extra checkout after the monolith has already decremented stock. In the Saga deployment, the original atomic `PlaceOrderUseCase` is split: `POST /orders` records a validated `DRAFT` and `POST /orders/{id}/checkout` starts distributed reservation/authorization/fulfillment. Never run both stock-decrement paths for the same order.

```text
POST /api/v1/orders/{orderId}/checkout
Idempotency-Key: customer-request-id

202 Accepted
Location: /api/v1/checkout-operations/{sagaId}
```

Checkout is asynchronous. The API returns an operation resource rather than holding an HTTP request open while services exchange messages.

```text
GET /api/v1/checkout-operations/{sagaId}

STARTED | IN_PROGRESS | CONFIRMED | REJECTED |
COMPENSATING | COMPENSATED | REQUIRES_ATTENTION
```

## Participants and ownership

| Participant | Owns | Does not own |
|---|---|---|
| Ordering | Draft validation, checkout status, final confirmation/rejection | Stock rows, payment authorization, fulfillment data |
| Inventory | Reserving/releasing requested copies with expiry | Order status or payment decision |
| Payment | Authorizing/voiding a payment token | Card data collection in this project, stock, order total calculation |
| Fulfillment | Creating/cancelling a fulfillment draft | Capturing money or deciding stock availability |
| Notification | Delivering confirmation/failure messages | Whether checkout succeeded |

The simulated payment service receives a provider token, amount, currency, and idempotency key. Card numbers and security codes never enter the bookstore services, events, logs, or test fixtures.

## Happy path

1. Ordering validates a `DRAFT` order and records `CHECKOUT_PENDING`.
2. Inventory reserves every requested book for a bounded period.
3. Payment authorizes the exact server-calculated total.
4. Fulfillment creates a cancellable draft shipment.
5. Ordering confirms the order—the Saga pivot.
6. Notification reacts after success and is retried independently.

## Business failure examples

- One title sold out before reservation.
- The payment provider declines the authorization.
- The address is outside all supported fulfillment regions.
- A reservation expires while payment is uncertain.
- A service accepts a command but its result message is delayed or duplicated.
- A compensation itself fails and requires operator intervention.

These are not all exceptions. Decline and out-of-stock are valid business outcomes; timeouts and unavailable brokers are technical uncertainty.

## Acceptance criteria shared by both variants

- Repeating the checkout request with the same idempotency key returns the same operation.
- Exactly one terminal customer decision is visible for an order.
- Every local state change and outgoing message is committed atomically through an outbox/publication log.
- Every consumer is idempotent through inbox/business keys.
- A failed forward path compensates completed compensable steps in reverse dependency order.
- Retries distinguish transient failure from business rejection.
- Deadlines survive restarts.
- Exhausted compensation reaches `REQUIRES_ATTENTION` with an operator action and audit trail.
- Notification failure never reverses a confirmed order.
- Trace/correlation data reconstructs the complete checkout timeline.

## Why not a distributed lock or 2PC

Long-lived locks across independently operated services reduce availability and are usually unsupported across their databases/providers. The business accepts a short eventual-consistency window and explicit compensation. If it could not, service separation would be the wrong design.

Continue to [Saga semantics and compensations](01-saga-semantics-and-compensations.md).

## Use, avoid, and operational notes

Use a Saga only after participants truly own separate transactions. Avoid it inside one database, where a local transaction is safer and faster. Measure checkout completion time, pending age, compensation rate, and manual-intervention rate—not only message throughput.
