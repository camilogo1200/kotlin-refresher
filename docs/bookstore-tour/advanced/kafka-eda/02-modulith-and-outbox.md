# Kafka 02 — Spring Modulith, publication registry, and outbox

## Business context

Before service extraction, modules need decoupled interaction and proof of boundaries. Later, the same committed business fact must be published externally without a database/Kafka dual-write gap.

## Spring Modulith bridge

Use Spring Modulith to verify application modules, test module interactions, and publish in-process application events. A module listener calls its own application use case; it does not read another module's tables.

## Reliability problem

This sequence is unsafe:

```text
commit order database
publish Kafka event
```

A crash between the operations loses the event. Reversing the order can publish an event for a transaction that later rolls back.

## Decision

Store a publication/outbox record in the same local transaction as the order. A separate publisher retries delivery and marks the record published. Compare:

- Spring Modulith event publication registry/externalization.
- An explicit relational outbox poller.
- CDC as an optional production variant.

Teach the failure model rather than presenting any annotation as atomic magic.

## Tests and operations

Kill the process after commit but before send, after send but before acknowledgement, and during republish. Prove eventual publication, possible duplicates, cleanup/retention, metrics for oldest unpublished record, and manual recovery.

