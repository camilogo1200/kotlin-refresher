# Event sourcing 03 — Trade-offs, snapshots, and evolution

## Business context

Event sourcing improves history and reconstruction but makes event contracts permanent operational assets. The team must evaluate its cost before treating it as the default persistence pattern.

## Costs

- Event schema evolution and upcasting.
- Replay-compatible code and fixtures.
- Projection lag and rebuild procedures.
- Expected-version conflicts and retry policy.
- Debugging state derived from many historical facts.
- Deleting/anonymizing regulated personal data in immutable history.
- Snapshot lifecycle if streams become long.

## Snapshots

Snapshots are a performance optimization, never the source of truth. Store aggregate id/version and serialized state, verify compatibility, then replay events after the snapshot version. Measure before adding them.

## Event evolution

Prefer additive changes and new event types. Never rewrite historical meaning casually. Upcasters/adapters translate old representations into the current in-memory event model while preserving original stored facts.

## Use/avoid decision

Use selectively where historical intent, audit, temporal queries, or rebuildable projections are central. Avoid for mostly static catalog reference data, simple CRUD, inexperienced teams without operational support, or requirements demanding immediate cross-model consistency.

## Completion test

The team can rebuild fulfillment state and projections from a clean database, explain every compatibility decision, and show why the benefit exceeds current-state persistence for this bounded context.

