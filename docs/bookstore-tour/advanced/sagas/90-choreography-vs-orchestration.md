# Saga 90 — Choreography versus orchestration

## Same business outcome

Both variants expose the same checkout operation, use the same participant use cases, apply the same compensations, and pass the same failure-injection suite. Only workflow ownership changes.

| Concern | Choreography | Orchestration |
|---|---|---|
| Control | Distributed across event reactions | Explicit coordinator/process manager |
| Messages | Primarily facts/events | Commands from coordinator plus result events |
| Simple workflow | Fewer moving parts | Coordinator may feel heavy |
| Growing branches | Dependencies become implicit | State machine remains visible |
| Participant coupling | Participants learn triggering facts | Participants know coordinator command contracts |
| Observability | Reconstructed across events | Coordinator state gives a direct view |
| Failure policy | Distributed among participants/order tracking | Centralized in coordinator |
| Availability risk | No coordinator component | Coordinator must be durable and replicated |
| Testing | Full flow needs many event reactions | State-machine paths are more deterministic |

## Recommendation for this bookstore

Implement choreography first to learn event-driven decoupling and its hidden process cost. Treat orchestration as the preferred final model once checkout includes inventory, payment, fulfillment, deadlines, multiple compensations, and operator recovery.

Keep simple broadcast facts—such as `OrderConfirmed` consumed independently by notification and analytics—choreographed even when checkout itself is orchestrated. A system can use both styles at different boundaries.

## Deployment comparison

```text
compose.saga-choreography.yml
compose.saga-orchestration.yml
```

Never enable both coordinators for the same topic/order population. Use distinct topic namespaces or environment isolation for the learning variants.

## Final design questions

1. Which action is the pivot, and why?
2. Which compensations can themselves fail?
3. How is an unknown payment outcome reconciled safely?
4. Who owns the checkout deadline in each variant?
5. What is the customer-visible state during compensation?
6. How does an operator recover without bypassing domain rules?
7. Which events are business audit and which are transport metadata?
8. Why does a Saga not guarantee immediate or exactly-once consistency?

Continue to [selective event sourcing and CQRS](../event-sourcing-cqrs/README.md), where fulfillment—not the entire bookstore—is evaluated for an event-sourced persistence model.

## Performance and industry note

Neither variant is inherently faster. Choreography removes a coordinator hop but can add distributed reactions and reconciliation; orchestration centralizes extra state and commands. Compare equivalent reliability, persistence, and compensation behavior under the same workload.
