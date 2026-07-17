# Kafka and event-driven architecture

The bookstore begins as a modular monolith. Kafka enters only when bounded contexts need independent deployment, scaling, or failure isolation.

## Lessons

1. [Why events now](00-from-modular-monolith-to-eda.md)
2. [Domain events and integration events](01-domain-events-and-integration-events.md)
3. [Spring Modulith and reliable publication](02-modulith-and-outbox.md)
4. [Kafka publisher contracts](03-kafka-publisher.md)
5. [One idempotent consumer](04-single-consumer.md)
6. [Multiple consumer groups](05-multiple-consumers-and-groups.md)
7. [Extracting bounded contexts](06-service-extraction.md)

The end state is not “Kafka everywhere.” Commands that require an immediate answer may remain HTTP; events communicate completed facts to independently interested consumers.

