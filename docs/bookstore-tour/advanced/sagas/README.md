# Distributed checkout Sagas

This track implements one business capability twice: checkout an existing draft order after inventory, payment, and fulfillment have separate ownership and databases.

## Why one use case, two variants

Choreography and orchestration must be compared against identical acceptance criteria, participant use cases, compensations, Kafka reliability guarantees, and failure tests. Creating two unrelated demos would hide the actual trade-off.

## Lessons

1. [Business context and customer contract](00-distributed-checkout-context.md)
2. [Saga semantics, pivot, and compensation](01-saga-semantics-and-compensations.md)
3. Choreography:
   - [Event flow](choreography/01-choreographed-checkout-flow.md)
   - [Participant implementation](choreography/02-participant-implementation.md)
   - [Failures, timeouts, and recovery](choreography/03-failures-timeouts-and-recovery.md)
4. Orchestration:
   - [Coordinator flow](orchestration/01-orchestrated-checkout-flow.md)
   - [Coordinator and participant implementation](orchestration/02-coordinator-implementation.md)
   - [Durability and recovery](orchestration/03-durability-and-recovery.md)
5. [Comparison and decision guide](90-choreography-vs-orchestration.md)

## Non-negotiable model

A Saga is a durable sequence of local transactions. It is not a distributed ACID transaction, not exactly-once delivery, not an in-memory state machine, and not a long-running `async` block. Compensations are new business actions; they do not rewind time.

The two variants run separately through different deployment compositions. They must never both react to the same checkout in one environment.

