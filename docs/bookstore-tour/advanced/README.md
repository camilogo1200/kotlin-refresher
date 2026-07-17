# Advanced architecture path

This phase begins only after the modular monolith and fulfillment API are independently correct.

## Progression

1. [Kafka and EDA](kafka-eda/README.md): internal domain facts become reliable external contracts.
2. [Sagas](sagas/README.md): distributed checkout is implemented twice, with choreography and orchestration.
3. [Event sourcing and CQRS](event-sourcing-cqrs/README.md): fulfillment persistence is selectively redesigned as an event-sourced experiment.

The advanced path adds operational cost intentionally: brokers, eventual consistency, duplicate delivery, ordering, schema evolution, tracing, compensation, and recovery. Every lesson must explain why the monolith or a local transaction is no longer enough.

