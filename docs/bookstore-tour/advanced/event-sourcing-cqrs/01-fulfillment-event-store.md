# Event sourcing 01 — Fulfillment event store and rehydration

## Business context

Operations need the exact fulfillment timeline, corrections, and reconstruction after projection bugs. A current-state row plus loosely governed audit entries cannot prove how the state was derived.

## Event stream

```text
FulfillmentStarted
BookPicked
PackagePrepared
CarrierAssigned
PackageCollected
InTransit
OutForDelivery
Delivered
DeliveryFailed
TrackingCorrectionRecorded
```

## Storage contract

Persist `eventId`, `aggregateId`, `aggregateType`, `streamVersion`, `eventType`, `schemaVersion`, timestamp, metadata, and serialized payload. Enforce uniqueness of `(aggregateId, streamVersion)` and `eventId`.

The repository loads ordered events, folds them through pure `apply` functions, asks the aggregate to handle a command, then appends new events only if the expected version still matches.

```text
events -> rehydrate aggregate -> handle command -> new events -> append atomically
```

Do not allow event handlers to call external systems during rehydration. Applying a historical event must be deterministic and side-effect free.

## Tests

Use given/when/then aggregate tests, expected-version conflict tests, unknown/old event version tests, and full-stream replay against production-like data samples.

## Use, avoid, and performance notes

Use per-aggregate streams when ordered history and optimistic expected-version writes match the domain. Avoid cross-aggregate transactions disguised inside one stream. Measure replay depth and append latency before adding snapshots or partitioning.
