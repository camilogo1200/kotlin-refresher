# Orchestration 02 — Coordinator and participant implementation

## Business context

A coordinator that keeps state in memory or publishes commands outside its transaction creates a new single point of inconsistency. It must be an ordinary durable application component with ports, persistence, and optimistic concurrency.

## Application service

```kotlin
/**
 * Durable process manager for distributed checkout.
 * It owns Saga progression and compensation policy, while participants own
 * their local business invariants and transactions.
 */
@Service
class CheckoutSagaCoordinator(
    private val sagas: CheckoutSagaRepository,
    private val commands: CheckoutCommandPublisher,
    private val clock: Clock,
)
```

The coordinator receives a start/reply input, loads the Saga, validates the expected state/message id, applies one transition, stores the new state plus outbox command in one local transaction, and returns. It never synchronously waits for another service.

## Ports and adapters

```text
StartCheckoutUseCase / HandleSagaReplyUseCase
  <- CheckoutSagaCoordinator @Service
      -> CheckoutSagaRepository
          <- JdbcCheckoutSagaRepository @Repository
      -> CheckoutCommandPublisher
          <- KafkaCheckoutCommandPublisher @Component
```

Participants use command listeners as inbound adapters and return result events through outboxes. Reuse the same `ReserveInventoryUseCase`, `AuthorizePaymentUseCase`, and compensation use cases from the choreographed variant; only coordination changes.

## Concurrency

Persist a Saga version and expected step. Two duplicate/concurrent replies cannot advance twice. Use message inbox uniqueness plus optimistic locking and retry the local coordinator transaction when appropriate.

## Deployment

The coordinator can run multiple instances because Kafka partitions and database optimistic locking serialize a Saga key. “Central” means centralized workflow policy, not a single unreplicated process.

## Use, avoid, and operational notes

Use a hand-written coordinator while workflow size remains understandable and the lesson benefits from visible mechanics. Evaluate a durable workflow engine when timers, branches, human tasks, and history grow substantially. Avoid an in-memory singleton coordinator.
