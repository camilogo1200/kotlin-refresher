# Kafka 03 — Publisher and topic contracts

## Business context

Fulfillment needs to start when an order is accepted, but ordering must not know the fulfillment deployment or wait for it synchronously.

## Contract

Publish a versioned `OrderPlacedV1` integration event from the outbox through `KafkaOrderEventPublisher @Component`. The application owns a `OrderEventPublisher` output port; Kafka types remain in the adapter.

Use a topic such as `bookstore.order-events.v1` and key records by `orderId` when per-order ordering matters. Document partition count, retention, producer acknowledgements, idempotent producer settings, serialization, and schema compatibility.

## Send semantics

Publication success means the broker acknowledged according to configured policy, not that every consumer completed. Handle asynchronous send result, timeout, retry, and outbox marking without blocking a request thread indefinitely.

## Tests and proof

Use Testcontainers Kafka to verify key, headers/envelope, serialization, partitioning assumption, outbox retry, and duplicate send after an uncertain acknowledgement.

## Don't

- Do not expose a domain entity directly as the message payload.
- Do not create one topic per entity id.
- Do not put secrets, payment tokens, or unnecessary customer PII in broad events.

## Performance and production note

Throughput depends on batching, compression, acknowledgements, partitions, record size, and broker capacity. Do not weaken durability settings merely to improve a local benchmark. Monitor send latency, error rate, retries, outbox age, and partition skew.
