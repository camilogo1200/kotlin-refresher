# Fulfillment 01 — Domain, use cases, and ports

## Business context

Carrier payloads and SSE representations change independently from fulfillment rules. The core needs a stable lifecycle and transition policy.

## Domain model

Use `Fulfillment`, `FulfillmentId`, `OrderId`, `TrackingNumber`, and past-tense `FulfillmentEvent` types. Define valid transitions explicitly; a package cannot move from `DELIVERED` back to `IN_TRANSIT` without a correction policy.

## Input ports and services

```text
StartFulfillmentUseCase       <- StartFulfillmentService @Service
RecordTrackingEventUseCase    <- RecordTrackingEventService @Service
GetFulfillmentQuery           <- GetFulfillmentService @Service
StreamFulfillmentEventsQuery  <- StreamFulfillmentEventsService @Service
```

Each `*Service` is the use-case interactor. Keep the same KDoc convention introduced in Core 05.

## Output ports

```kotlin
interface FulfillmentRepository {
    suspend fun find(id: FulfillmentId): Fulfillment?
    suspend fun save(fulfillment: Fulfillment): Fulfillment
}

interface FulfillmentEventRepository {
    suspend fun append(event: FulfillmentEvent)
    fun stream(id: FulfillmentId): Flow<FulfillmentEvent>
}
```

`Flow` appears only on the stream contract. Ordinary current-state reads return a suspended value.

## Tests

Plain tests cover state transitions and duplicate event identity. Application tests prove mapping, port calls, cancellation, and that rejected transitions are not persisted.

## Performance and production note

Keep event streams keyed and bounded by fulfillment id; a global unfiltered stream can create memory and authorization risk. Domain transition checks are cheap—database queries, stream fan-out, and retained history are the capacity drivers.
