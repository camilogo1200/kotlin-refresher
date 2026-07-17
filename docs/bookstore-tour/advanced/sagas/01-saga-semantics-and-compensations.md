# Saga 01 — Steps, pivot, retries, and compensations

## Problem

“Rollback the order” is not a sufficient failure policy. Each participant has already committed locally, and some real-world effects cannot be erased. The workflow needs explicit forward, compensation, and recovery semantics.

## Step classification

| Step | Local result | Classification | Compensation/recovery |
|---|---|---|---|
| Validate/start checkout | `CHECKOUT_PENDING` | Compensable | Mark rejected/cancelled |
| Reserve inventory | Reservation with expiry | Compensable | Release reservation |
| Authorize payment | Authorization, not card capture | Compensable | Void authorization |
| Create fulfillment draft | Cancellable draft | Compensable | Cancel fulfillment |
| Confirm order | Customer purchase accepted | **Pivot** | No automatic rollback in this Saga |
| Notify customer | Delivery attempt | Retryable side effect | Retry/DLT/manual delivery; never unconfirm |

The pivot is the point after which the workflow must move forward through retryable actions rather than attempt to pretend the purchase never happened.

## Compensation is business behavior

`ReleaseInventory`, `VoidPaymentAuthorization`, and `CancelFulfillment` are explicit idempotent use cases with their own validation, authorization, audit, outbox, tests, and failure states. Compensation might restore resource availability, but it does not delete history.

## Failure classification

- **Business rejection:** out of stock, payment declined, unsupported destination. Do not retry unless business input changes; start compensation.
- **Transient technical failure:** timeout, broker unavailable, database deadlock. Retry with bounded backoff/jitter while the deadline permits.
- **Unknown outcome:** request may have succeeded but reply was lost. Query/deduplicate using command id before repeating a side effect.
- **Permanent technical failure:** invalid schema, incompatible event, corrupted state. Stop automatic retry and route to intervention.

## Shared identifiers

Every command/event carries `messageId`, `sagaId`, `orderId`, `correlationId`, `causationId`, `schemaVersion`, and timestamp. Commands also carry an idempotency key and expected business amount/version where needed.

## Durable time

Never implement a business deadline with only `delay()` or `withTimeout`. Persist `deadlineAt` and resume timeout evaluation after restart. Reservation expiry is owned by inventory; checkout deadline is owned by ordering/choreographer or the orchestrator, depending on the variant.

## Coroutines boundary

Coroutines remain valuable inside one message handler for local concurrent IO. They do not own the cross-service workflow lifetime:

```kotlin
// Wrong: process memory cannot own a multi-minute distributed Saga.
async {
    reserveInventory()
    authorizePayment()
    createFulfillment()
}
```

The durable state is in databases/outboxes and Kafka; no `Deferred<T>` survives a process crash.

## Shared verification matrix

Both variants must inject failure before and after every local commit and message acknowledgement. Test duplicate, delayed, missing, and out-of-order messages; process restart; deadline; compensation retry; and manual resolution.

Choose [choreography](choreography/01-choreographed-checkout-flow.md) first, then implement the orchestration alternative.

## Use, avoid, and production notes

Use compensation where the business can define an acceptable counter-action. Avoid calling a technical delete a compensation or promising rollback of irreversible physical effects. Keep compensation latency, retry age, and unresolved unknown outcomes visible to operators.
