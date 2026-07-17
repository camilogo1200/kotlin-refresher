# Choreography 01 — Checkout event flow

## Business context

The first Saga variant avoids a central coordinator. Each participant subscribes to a completed fact, commits its own action, and publishes the next fact. This fits a short, stable workflow and makes EDA mechanics visible.

## Happy path

```text
OrderCheckoutStarted
    -> Inventory reserves
InventoryReserved
    -> Payment authorizes
PaymentAuthorized
    -> Fulfillment creates draft
FulfillmentPrepared
    -> Ordering confirms
OrderConfirmed
    -> Notification sends independently
```

Events describe facts. `InventoryReserved` is not a disguised command telling payment how to implement its work; payment chooses to react because that fact satisfies its precondition.

## Rejection paths

```text
InventoryRejected
    -> Ordering rejects checkout

PaymentDeclined
    -> Inventory releases reservation
InventoryReleased
    -> Ordering marks checkout compensated/rejected

FulfillmentRejected
    -> Payment voids authorization
    -> Inventory releases reservation
PaymentAuthorizationVoided + InventoryReleased
    -> Ordering marks checkout compensated/rejected
```

Ordering tracks the externally visible checkout operation and which required compensation outcomes have arrived. It does not send every forward command; participant reactions remain distributed.

## Timeout path

Ordering persists a checkout deadline. If no terminal forward result arrives, it publishes `CheckoutTimedOut`. Participants inspect their local state and idempotently compensate what they actually completed, then publish compensation results.

## Coupling to make visible

Choreography removes a coordinator but not coupling. Each participant knows which facts trigger it, and the complete workflow is distributed across handlers. Adding steps can create event cycles and hidden dependencies. Maintain an event-flow document and trace view as executable operational artifacts.

## Exit criteria

Every event has one owner, trigger conditions are documented, and the happy/rejection/timeout flows reach one customer-visible terminal state.

## Use, avoid, and performance notes

Use choreography for a small stable participant graph and independent reactions. Avoid it when branching and recovery become hard to see. End-to-end latency is the sum of broker, queueing, handler, outbox, and retry delays across every hop.
