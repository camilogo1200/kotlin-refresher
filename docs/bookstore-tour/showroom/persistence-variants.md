# Showroom — Persistence variants

## Problem

Different bookstore capabilities have different storage needs. A prototype needs speed of development, inventory reporting needs explicit SQL, ordering needs aggregate consistency, and a streaming API may need non-blocking IO.

## Variants

| Variant | Best lesson/use | Avoid when |
|---|---|---|
| Pure Kotlin in-memory adapter | Port contract tests, prototypes, deterministic demos | Durability, multiple instances, realistic concurrency |
| Spring `JdbcClient` | Explicit SQL, projections, batch inventory operations | You want repository-derived aggregate CRUD |
| Spring Data JDBC | Explicit aggregates and straightforward relational persistence | Rich ORM navigation/unit-of-work behavior is central |
| JPA/Hibernate | Existing ORM-heavy systems, complex mapping ecosystem | You want the simplest Kotlin-first aggregate model |
| Spring Data R2DBC | End-to-end non-blocking workloads and streams | Drivers/libraries block or workload is normal CRUD |

## Adapter swap exercise

Keep `BookRepository` and its contract tests stable. Implement an in-memory and JDBC adapter. Select one explicitly through a profile; do not inject both accidentally.

For R2DBC, use the fulfillment API rather than forcing JDBC and R2DBC repositories into the same beginner application. The application port may remain `suspend`/`Flow`, while the adapter and transaction mechanism change.

## Performance notes

JDBC consumes a thread and connection while waiting. Virtual threads reduce platform-thread cost but not connection usage. R2DBC removes thread-per-wait at the driver level but adds reactive transaction/context requirements. In-memory timing says nothing about production database behavior.

