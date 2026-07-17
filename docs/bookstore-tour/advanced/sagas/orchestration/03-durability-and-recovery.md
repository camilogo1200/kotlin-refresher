# Orchestration 03 — Durability, deadlines, and recovery

## Business context

The orchestrator must recover after restarting between any two instructions, including after a local commit but before Kafka acknowledgement. Operators need a complete view and a safe way to resume or compensate.

## Persisted Saga record

Store at least:

```text
sagaId, orderId, state, version
completedSteps and compensation status
currentCommandId / expectedReplyType
deadlineAt, nextAttemptAt, retryCount
lastErrorCode and classification
createdAt, updatedAt
```

Avoid persisting arbitrary framework objects or serialized coroutine continuations. The state schema is a business process contract and needs migrations.

## Recovery loop

- A scheduler queries due `nextAttemptAt`/expired deadlines.
- It locks/transitions a Saga atomically and writes the next command to outbox.
- Unknown outcomes use participant query/idempotency before repeating effects.
- Compensation proceeds until all completed compensable steps are confirmed undone.
- Exhausted recovery transitions to `REQUIRES_ATTENTION` with a controlled operator command.

## Operator actions

Provide authenticated, audited operations such as retry current step, re-query participant, begin compensation, or mark a verified external outcome. Never provide a generic “set Saga state” endpoint.

## Tests

Run the same failure matrix as choreography plus coordinator-specific cases: duplicate reply, reply for wrong state, optimistic conflict, restart after transition before send, concurrent timeout/reply, compensation resume, and operator retry.

## Performance

The orchestrator adds writes and a central policy component but improves visibility. Partition commands by `sagaId`; size coordinator instances from command rate, database/outbox capacity, and reply latency, not customer HTTP concurrency.

## Use and avoid

Use operator actions that execute audited domain/application commands. Avoid direct database state edits, generic force-complete endpoints, and automatic retries after an outcome becomes genuinely unknown without first reconciling the participant.
