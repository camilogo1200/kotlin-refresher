# Kafka 04 — One idempotent consumer

## Business context

The fulfillment API must start work for every accepted order. Kafka delivery is at least once from the application's perspective, so duplicates and redelivery after crashes are normal.

## Adapter flow

```text
OrderPlacedKafkaConsumer @Component
  -> StartFulfillmentUseCase
      <- StartFulfillmentService @Service
          -> FulfillmentRepository
```

The listener validates the envelope, maps to a command, and delegates. It owns deserialization and acknowledgement concerns, not fulfillment rules.

## Idempotency

Persist the consumed `eventId` in an inbox record in the same local transaction as the fulfillment change. A unique constraint turns duplicate delivery into a no-op. Business idempotency by `orderId` is also required; transport deduplication alone is not enough.

## Failure policy

- Retry transient database/network failures with bounded backoff.
- Do not retry permanent schema or business rejections forever.
- Route poison messages to a dead-letter path with diagnostics and replay procedure.
- Preserve cancellation/shutdown and avoid acknowledging before local commit.

## Tests

Prove duplicate delivery, crash before commit, crash after commit before acknowledgement, malformed event, exhausted retry, dead-letter routing, and safe replay.

## Use, avoid, and performance notes

Use a consumer for asynchronous work that can tolerate eventual completion. Avoid it when the caller requires an immediate authoritative answer. Handler latency, database transaction time, partition count, retry delay, and downstream capacity determine consumer throughput.
