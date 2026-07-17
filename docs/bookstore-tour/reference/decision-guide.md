# Technology and architecture decision guide

| Choice | Use when | Avoid when |
|---|---|---|
| Simple controller/service/repository | Small CRUD and little business policy | Technology leaks or multiple adapters/use cases are growing |
| Feature-first hexagonal | Long-lived rules, multiple adapters, testing/replaceability matter | One trivial table does not justify mapping/files |
| Spring Data JDBC | Explicit aggregates and simple relational persistence | Rich ORM navigation/unit-of-work is required |
| `JdbcClient` | Explicit SQL/projections/batches | Repository abstraction already fits well |
| MVC + JDBC | Conventional request/response API with blocking libraries | Many long-lived streams and end-to-end reactive dependencies |
| MVC + virtual threads | Blocking stack needs cheaper waits with low rewrite cost | Connection/downstream capacity is assumed infinite |
| WebFlux + R2DBC | High-concurrency streaming path is non-blocking end to end | Normal CRUD or blocking dependencies dominate |
| Coroutines | Structured concurrency, cancellation, readable async composition | Used to hide unavoidable blocking without capacity policy |
| Kafka/EDA | Independent consumers, scale/release/failure isolation | Simple synchronous request-response is sufficient |
| Choreographed Saga | Short stable workflow with few participants | Branching, deadlines, compensations, and visibility are complex |
| Orchestrated Saga | Multi-step process needs explicit durable policy/recovery | A local transaction or simple event reaction solves the problem |
| Event sourcing | History/reconstruction and projections justify permanent event model | Simple current-state CRUD or operational maturity is absent |

Always record business consistency, failure, security, and operator-recovery requirements before selecting technology.

