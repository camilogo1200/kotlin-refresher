# Core 12 — Observability, testing, and release readiness

## Business context

A correct API that cannot be diagnosed, deployed, or changed safely is not production-ready. Operators need request correlation and health signals; developers need fast tests at every ring.

## Observability

- Generate or accept a request/correlation id at the input adapter.
- Use parameterized structured logs and clear MDC state reliably.
- Verify context after coroutine suspension; do not assume raw `ThreadLocal` survives.
- Expose only intended Actuator health, info, and metrics endpoints.
- Record outbound latency, database pool pressure, optimistic conflicts, and order outcomes.
- Never put payment tokens, emails, or sensitive event payloads into unrestricted logs.

## Test pyramid

| Ring | Test |
|---|---|
| Domain | Plain JUnit/Kotlin, no Spring or mocks |
| Application | `runTest` with fakes/MockK ports |
| HTTP adapter | `@WebMvcTest` contract and error tests |
| JDBC adapter | `@DataJdbcTest` plus Testcontainers PostgreSQL |
| Full slice | `@SpringBootTest` for critical order workflow |
| Architecture | ArchUnit dependency and annotation policy |

Performance tests must state workload, environment, warmup, concurrency, and limiting resource. Do not turn a local timing into an industry throughput claim.

## Release checklist

- Flyway migrations are forward-tested and rollback strategy is documented.
- Configuration is immutable, validated, and supplied through environment-specific sources.
- Timeouts, pool sizes, dispatcher limits, and downstream bulkheads are aligned.
- Problem Details do not leak internals.
- Health checks distinguish process liveness from dependency readiness.
- Critical concurrent and cancellation paths have repeatable tests.
- API examples and operational commands match the running application.

## Next paths

The core monolith is complete. Choose the [technology showroom](../showroom/README.md) for comparisons or build the [WebFlux fulfillment API](../webflux-fulfillment/README.md) before entering Kafka and distributed workflows.

## Industry practice

Use production-like integration tests for database/broker semantics and fast unit tests for policy. Avoid making every test a full-context test or treating logs as the only audit. Test and observability volume have real runtime/storage cost; retain high-value signals with controlled cardinality.
