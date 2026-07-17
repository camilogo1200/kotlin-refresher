# Kafka 01 — Domain events and integration events

## Business context

The ordering aggregate can raise `OrderPlaced`, but internal domain facts are not automatically stable public contracts. External consumers deploy independently and cannot be forced to understand internal classes.

## Distinction

| Type | Owner and purpose |
|---|---|
| Domain event | Internal past-tense fact supporting domain/module behavior |
| Integration event | Versioned external contract designed for independent consumers |
| Command | Request for one owner to perform an action; may be rejected |

Map domain events to integration events at an outbound adapter boundary. Include only data consumers are allowed and expected to use.

## Event envelope

```text
eventId
eventType
schemaVersion
occurredAt
producer
aggregateId/orderId
correlationId
causationId
payload
```

Do not use Kotlin/Jackson class names as long-term topic contracts. Define compatibility rules, ownership, retention, classification, and deprecation before the first independent consumer.

## Tests

Use contract tests and serialization golden samples. Prove old consumers tolerate additive fields and that unsupported versions fail observably rather than being silently misread.

## Use, avoid, and operational notes

Publish an integration event when multiple independent consumers need a committed fact. Avoid publishing every internal field change. Smaller stable payloads reduce bandwidth and coupling, but key-only events can cause synchronous callback traffic; choose intentionally.
