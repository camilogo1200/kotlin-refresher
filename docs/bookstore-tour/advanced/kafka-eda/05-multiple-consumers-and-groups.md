# Kafka 05 — Multiple consumers, groups, ordering, and concurrency

## Business context

After fulfillment starts, email, analytics, and audit projections also need the same event. Fulfillment itself needs multiple instances for throughput and availability.

## Group semantics

```text
Same group id:
  fulfillment-1 + fulfillment-2 divide partitions/work

Different group ids:
  fulfillment, notification, analytics each receive the event
```

Use distinct group ids for independent business consumers. Instances of the same consumer type share one group.

## Ordering and capacity

Kafka orders records within a partition, not across a topic. Key by aggregate/order id when that ordering is required. Maximum useful parallelism for one consumer group is bounded by assigned partitions. More listener threads than partitions do not create more ordered work.

Non-blocking retry topics can change effective ordering. Decide whether per-order ordering, throughput, or delayed retry is more important; document the trade.

## Operations

Measure consumer lag, processing latency, retries, DLT rate, rebalance time, partition skew, and inbox conflicts. Scale from observed lag and handler capacity, not CPU alone.

## Tests

Prove fan-out across groups, load sharing within a group, per-key ordering, rebalance recovery, duplicate idempotency, and a deliberately slow consumer that does not block other groups.

## Use and avoid

Use separate groups for independent business outcomes and one group for replicated instances of the same outcome. Avoid creating a new group for every deployment instance or assuming more threads than partitions increase ordered throughput.
