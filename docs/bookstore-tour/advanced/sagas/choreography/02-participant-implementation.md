# Choreography 02 — Participant implementation

## Business context

Kafka listeners are transport adapters. If reservation, payment, or compensation rules live inside them, the same behavior cannot be reused from HTTP/admin recovery or tested without Kafka.

## Handler shape

```text
Kafka listener @Component
  -> participant input port
      <- participant application service @Service
          -> repository/output ports
              <- local adapters @Repository/@Component
```

Example:

```kotlin
@Component
class CheckoutStartedConsumer(
    private val reserveInventory: ReserveInventoryUseCase,
) {
    @KafkaListener(topics = ["bookstore.checkout-events.v1"])
    suspend fun consume(event: OrderCheckoutStartedV1) {
        reserveInventory.execute(event.toCommand())
    }
}
```

The local application service performs one atomic local transaction that stores participant state, inbox/idempotency data, and an outbox result. Publication occurs later/reliably.

## Event contracts

Keep forward and compensation outcomes explicit. Use a shared envelope but participant-owned payload schemas. Avoid one enormous `CheckoutEvent` with dozens of nullable fields and a type switch.

## Idempotency rules

- Inventory reservation unique by `sagaId`/order line set.
- Payment authorization unique by provider idempotency key.
- Fulfillment draft unique by order/saga.
- Compensation safe when the forward step never happened or was already compensated.
- Ordering terminal transition guarded by optimistic version and valid state.

## Testing

Unit-test each application service without Kafka. Contract-test serialization. Use Kafka/Testcontainers integration tests for adapter acknowledgement, duplicates, partition key, and outbox publication.

## Don't

- Do not call the next service directly from the listener.
- Do not update another service's database.
- Do not treat event receipt order as globally guaranteed.

## Performance and production note

Keep listener transactions short and bound concurrency to database/downstream capacity. Inbox/outbox writes add intentional IO; benchmark with them enabled because removing reliability mechanisms changes the contract being measured.
