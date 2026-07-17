# Kafka 00 — From modular monolith to EDA

## Business context

Ordering, fulfillment, notification, and analytics now have different release and scaling needs. Direct module calls make failures and ownership increasingly coupled, but splitting everything at once would replace simple local transactions with distributed uncertainty.

## Request

Evolve in stages:

```text
1. Module boundaries inside one process
2. In-process domain events
3. Durable publication/outbox
4. Kafka integration events
5. Independent consumers
6. Extract one bounded context at a time
```

## Acceptance criteria

- Ordering commits its local truth without requiring notification or analytics availability.
- External events are not lost between database commit and broker publication.
- Consumers tolerate duplicates and restart safely.
- Eventual consistency and stale-read windows are documented.
- A synchronous request is not converted to messaging without a business reason.

## Do and don't

- **Do:** split by domain ownership and operational need.
- **Don't:** use a broker to avoid defining service contracts.
- **Don't:** assume asynchronous means faster or simpler.
- **Don't:** expose personal data in broadly consumed events without an explicit policy.

