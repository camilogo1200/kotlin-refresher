# Kafka 06 — Extracting bounded contexts

## Business context

Module boundaries and event contracts are proven. Fulfillment now needs independent scaling and releases, so it becomes the first extracted service. Notification and analytics can follow because their failure must not block ordering.

## Extraction sequence

1. Verify the bounded context owns its data and use cases.
2. Stop direct cross-module table access.
3. Establish an integration event and reliable publication.
4. Run the new service in shadow/read-only mode if useful.
5. Switch the inbound adapter from internal event/HTTP to Kafka.
6. Monitor lag, correctness, and reconciliation before deleting the old path.

## Target landscape

| Component | Owns |
|---|---|
| Bookstore/order API | Catalog and order acceptance |
| Fulfillment API | Fulfillment state, carrier events, live stream |
| Notification worker | Delivery attempts/templates, not order truth |
| Analytics worker | Replaceable projections |

Each service owns its datastore. Shared libraries contain only stable primitives/contracts and must not recreate a distributed monolith through shared internal models.

## Next pressure

Checkout now spans order, inventory, payment, and fulfillment ownership. A local transaction cannot coordinate it. Continue to the [Saga track](../sagas/README.md).

## Use, avoid, and operational notes

Extract when independent ownership, release cadence, scale, or fault isolation outweigh network and consistency cost. Avoid service-per-table and framework-driven splits. Compare end-to-end latency, operational staffing, broker/database cost, and reconciliation burden with the modular monolith baseline.
