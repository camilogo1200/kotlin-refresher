# Choreography 03 — Failures, timeouts, and recovery

## Business context

With no coordinator, no single process automatically knows whether a missing event means slow work, a lost publication, a crashed consumer, or an incompatible message. Recovery must be designed into each participant and the checkout operation view.

## Recovery mechanisms

- Ordering persists deadline and observed milestone/compensation events.
- Every participant exposes an operator-safe query by `sagaId` for reconciliation.
- Outbox lag and consumer lag are monitored separately.
- Duplicate compensation is a successful no-op.
- DLT records retain correlation, error classification, original topic/partition/offset, and replay policy.
- A reconciler can compare ordering's expected milestones with participant status and republish a safe command/event only after identifying ownership.

## Failure scenarios

1. Inventory commits but `InventoryReserved` is delayed: outbox republishes; checkout remains pending.
2. Payment authorizes but reply is lost: idempotency/query prevents a second charge; republish result.
3. Fulfillment rejects: payment void and inventory release occur independently; ordering waits for both outcomes.
4. One compensation exhausts retries: checkout becomes `REQUIRES_ATTENTION`, not falsely `REJECTED`.
5. Late success arrives after timeout: participant state policy determines whether it is ignored and compensated; never reconfirm silently.

## Observability

Build a timeline by `sagaId` containing every event id, causation, participant status, retries, deadlines, and terminal decision. Distributed traces help live diagnosis, but persisted business audit must not depend solely on trace retention.

## Choreography suitability test

If adding a participant requires many existing services to learn new event branches, or operators cannot answer “what happens next?” from one model, the workflow has outgrown choreography. Continue to [orchestration](../orchestration/01-orchestrated-checkout-flow.md).

## Use, avoid, and operational notes

Use automated reconciliation for known, idempotent recovery rules. Avoid blind event replay against side effects. Monitor oldest pending checkout, unmatched milestones, outbox/consumer lag, DLT age, and compensations awaiting confirmation.
