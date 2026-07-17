# Orchestration 01 — Coordinator-controlled checkout

## Business context

Checkout now has several participants, ordered compensations, deadlines, and an operator requirement to see the next expected action. A central process manager makes workflow policy explicit while participant services retain their local business rules.

## Command/reply flow

```text
CheckoutSagaCoordinator
  -> ReserveInventoryCommand
  <- InventoryReserved | InventoryRejected
  -> AuthorizePaymentCommand
  <- PaymentAuthorized | PaymentDeclined
  -> PrepareFulfillmentCommand
  <- FulfillmentPrepared | FulfillmentRejected
  -> ConfirmOrderCommand
  <- OrderConfirmed
```

On failure, the coordinator sends explicit compensation commands in reverse completed order:

```text
CancelFulfillmentCommand
VoidPaymentAuthorizationCommand
ReleaseInventoryCommand
RejectOrderCommand
```

## Responsibility boundary

The coordinator owns process sequence, deadline, retry/compensation decision, and Saga state. It does not decide whether stock exists, whether payment is acceptable, or whether an address is serviceable. Those invariants remain with participants.

## State model

```text
STARTED
INVENTORY_PENDING -> INVENTORY_RESERVED
PAYMENT_PENDING -> PAYMENT_AUTHORIZED
FULFILLMENT_PENDING -> FULFILLMENT_PREPARED
CONFIRMATION_PENDING -> COMPLETED

any pre-pivot failure
  -> COMPENSATING
  -> COMPENSATED | REQUIRES_ATTENTION
```

Only valid expected replies advance state. Duplicate/late replies are idempotently recorded and ignored or reconciled according to policy.

## Use, avoid, and performance notes

Use orchestration when process policy, branching, deadlines, and recovery need one explicit model. Avoid letting the coordinator absorb participant domain rules. The coordinator adds hops and writes; size it from Saga transition rate and persistence/outbox capacity.
