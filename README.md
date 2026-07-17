# 📚 Bookstore API — Kotlin-first Spring Boot refresher

![Kotlin](https://img.shields.io/badge/Kotlin-2.3.21-7F52FF?logo=kotlin&logoColor=white)
![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.0-6DB33F?logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-4169E1?logo=postgresql&logoColor=white)
![Architecture](https://img.shields.io/badge/Architecture-Hexagonal-blueviolet)

This repository is a guided build of a bookstore catalog and order-intake API. Its main purpose is to refresh **Kotlin first**, then apply Kotlin's concurrency model to an idiomatic annotated Spring Boot API organized with **feature-first hexagonal architecture**.

> **Current status:** the repository contains the generated Spring Boot skeleton and local infrastructure. The bookstore domain, endpoints, migrations, coroutine dependencies, adapters, and full test suite are learning milestones described in the docs; they are not implemented yet.

- [`docs/bookstore-guided-tour.md`](docs/bookstore-guided-tour.md) — curriculum hub linking the split, context-first lessons
- [`docs/bookstore-tour/README.md`](docs/bookstore-tour/README.md) — complete core, showroom, WebFlux, Kafka, Saga, and event-sourcing map
- [`docs/bookstore-api-sdd.md`](docs/bookstore-api-sdd.md) — scope, architecture decisions, package policy, and definition of done

## Learning path

The curriculum deliberately starts outside Spring:

1. **K0 — Kotlin modeling:** nullability, immutability, `data class`, regular class, `value class`, sealed results, collections, and extension functions.
2. **K1 — Structured concurrency:** `suspend`, `CoroutineScope`, `Job`, `Deferred<T>`, `launch`, `async`, `join`, `await`, and `awaitAll`.
3. **K2 — Context and streams:** cancellation, supervision, dispatcher ownership, `withContext`, cold `Flow`, and deterministic tests with `runTest`.
4. **Spring milestones:** annotated MVC controllers, use cases, ports/adapters, Data JDBC, transactions, external HTTP, resilience, observability, and API tests.
5. **Runtime comparisons:** virtual threads for imperative blocking code and R2DBC for truly non-blocking relational IO.
6. **Reactive companion API:** fulfillment and live tracking through WebFlux, R2DBC, Kotlin `Flow`, and SSE.
7. **Advanced architecture:** Spring Modulith events, outbox, Kafka, service extraction, choreographed/orchestrated Sagas, and selective event sourcing/CQRS.

The main line remains coroutine-shaped. Comparison branches are there to make trade-offs concrete, not to replace the Kotlin-first path.

## Target problem

The API will be the system of record for a small catalog and order workflow:

- enforce ISBN, money, stock, and order-total invariants in pure Kotlin;
- resolve concurrent purchases of the final copy with optimistic locking: one commit and one `409 Conflict`;
- expose stable, versioned HTTP contracts and RFC 9457 `ProblemDetail` errors;
- verify prices through an outbound HTTP port before opening the short database transaction;
- keep request correlation, health, metrics, migrations, and architecture tests visible.

Payments, shipping, taxes, authentication/authorization, messaging, and a frontend remain outside the **core monolith's production scope**. The advanced curriculum adds simulated payment authorization, fulfillment, Kafka, and distributed workflows as isolated learning applications; it never handles card data or pretends those additions are required for the core API.

## Technology direction

| Concern | Main line | Comparison or later milestone |
|---|---|---|
| Language/runtime | Kotlin 2.3.21, Java 21 | — |
| Framework | Spring Boot 4.1.0 / Spring Framework 7 | — |
| Web | Annotated Spring MVC with `suspend` handlers and `Flow` where the HTTP contract streams | Imperative MVC on virtual threads |
| Persistence | Spring Data JDBC; immutable persisted rows; Flyway-owned schema | Spring Data R2DBC adapter |
| Database | PostgreSQL 18 through Docker Compose | — |
| Outbound HTTP | Spring HTTP interface backed by `WebClient` for suspend APIs | `RestClient` for the blocking comparison |
| Concurrency | Structured coroutines, injected dispatcher policy, adapter-owned blocking bridge | `CompletableFuture`/virtual-thread comparison |
| Testing | Kotlin test/JUnit Platform, `kotlinx-coroutines-test`, MockK, MVC/Data JDBC slices, Testcontainers, ArchUnit | R2DBC adapter tests on its branch |
| Reactive companion | — | WebFlux fulfillment API, R2DBC, `Flow`, SSE, carrier webhooks |
| Messaging/distribution | — | Spring Modulith, outbox, Kafka, consumer groups, Sagas, optional event sourcing/CQRS |

The existing build already has Spring MVC, Spring Data JDBC/JDBC, Actuator, PostgreSQL, Docker Compose support, `plugin.spring`, and strict Kotlin compiler options. The guided milestones add Flyway, validation, coroutine, resilience, and testing dependencies when they first become useful. `plugin.jpa` is currently present in the generated build but is not required by the Data JDBC main line; the guide treats JPA as an optional comparison only.

## Architecture

The project is a modular monolith organized **by feature first** and **by hexagonal role inside each feature**. Dependencies point inward: HTTP and persistence adapters depend on application ports and domain types, never the reverse.

```text
com.example.kotlinrefresher
├── bootstrap/                         # app entry point and cross-cutting Spring configuration
├── catalog/
│   ├── domain/                        # pure Kotlin model, rules, domain errors
│   ├── application/
│   │   ├── port/input/                # use-case contracts
│   │   ├── port/output/               # repository/external contracts required by the core
│   │   └── usecase/                   # orchestration implementations
│   └── adapter/
│       ├── input/http/                 # controllers, request/response DTOs
│       └── output/persistence/jdbc/    # Data JDBC rows, repositories, mappings, adapters
├── ordering/
│   ├── domain/
│   ├── application/port/input/
│   ├── application/port/output/
│   ├── application/usecase/
│   ├── adapter/input/http/
│   ├── adapter/output/persistence/jdbc/
│   └── adapter/output/pricing/
└── shared/                             # only genuinely shared concepts; not a dumping ground
```

`input` and `output` are used instead of `in` and `out` because `in` is a Kotlin keyword. Ports exist at architectural boundaries; there is no automatic `FooService`/`FooServiceImpl` pair for every class.

### Spring annotation policy

Spring annotations are intentional and semantic:

| Location | Annotation | Meaning |
|---|---|---|
| `adapter/input/http` | `@RestController` | inbound HTTP adapter |
| `application/usecase` | `@Service` | application orchestration/use case |
| `adapter/output/persistence` | `@Repository` | persistence role and scan boundary; Spring Data proxies translate JDBC failures |
| other output adapters/infrastructure | `@Component` | generic Spring-managed adapter |
| `bootstrap` | `@Configuration`, `@Bean`, `@ConfigurationProperties` | composition and typed runtime configuration |
| `domain` | none | framework-free Kotlin business model |

Spring Data repository interfaces are discovered automatically; adding `@Repository` to them only for decoration is unnecessary. The custom adapter implementing an output port is the meaningful `@Repository` boundary.

The later curriculum adds separate deployment units rather than mixing incompatible runtime lessons into the core application:

```text
bookstore-api              MVC + JDBC modular monolith: catalog and ordering
bookstore-fulfillment-api  WebFlux + R2DBC: shipment timeline and live tracking
notification-worker       Kafka consumer: customer delivery attempts
analytics-worker          Kafka consumer: replaceable projections
checkout-orchestrator     Optional durable coordinator for the orchestrated Saga variant
```

## Coroutine and transaction rules

- Controllers and asynchronous input/output ports may be `suspend`; domain behavior is ordinary synchronous Kotlin.
- `async` is created inside `coroutineScope` or `supervisorScope`, never in `GlobalScope`.
- `Deferred.join()` waits and returns `Unit`; `await()` returns the value and surfaces failure; `awaitAll()` is used when fail-fast group semantics are intended.
- `withContext` belongs in an output adapter when bridging a blocking library. It does not make JDBC non-blocking.
- A Data JDBC transaction remains imperative and on one thread. Suspend orchestration completes remote calls first, then invokes a short `@Transactional` method on a separate proxied bean.
- `Flow` is used only when the contract benefits from laziness, streaming, or composition. Ordinary CRUD lists stay paged.

## Getting started

Prerequisites: JDK 21+ and Docker Desktop/Engine.

The current skeleton has two pre-M0 infrastructure gaps that are deliberately not hidden by this README:

- `.env.example` does not define `REDIS_IMAGE` even though `compose.yaml` references it;
- the PostgreSQL `environment` block uses list items containing maps, which Docker Compose rejects (`unexpected type map[string]interface {}`).

Core 03 in the split curriculum fixes those entries (or removes the out-of-scope Redis service) and verifies `docker compose --env-file .env.example config --quiet`. After that reconciliation, the normal commands are:

```powershell
Copy-Item .env.example .env
./gradlew.bat bootRun
```

Run the current test suite with:

```powershell
./gradlew.bat test
```

At the current skeleton stage, the single test only checks application-context startup; it is currently gated by the Compose issues above and by dependency availability. The API routes below are target milestones, not currently available. Redis is local scaffolding, not part of the bookstore learning path.

## Planned API

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/v1/books` | create a book |
| `GET` | `/api/v1/books/{id}` | retrieve a book |
| `PUT` | `/api/v1/books/{id}` | update a book |
| `DELETE` | `/api/v1/books/{id}` | delete a book |
| `GET` | `/api/v1/books` | paged, sorted, filtered catalog listing |
| `POST` | `/api/v1/orders` | verify prices and place an order atomically |
| `GET` | `/api/v1/orders/{id}` | retrieve an order aggregate |
| `POST` | `/api/v1/orders/{id}/checkout` | advanced: start distributed checkout and return `202` operation resource |
| `GET` | `/api/v1/checkout-operations/{sagaId}` | advanced: inspect checkout/Saga progress |

The Saga deployment is an alternative evolution: it refactors core order placement into draft creation plus asynchronous checkout. It does not execute distributed checkout after the core transaction has already decremented stock.

The companion fulfillment API adds:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/v1/fulfillments` | start fulfillment before the Kafka adapter is introduced |
| `POST` | `/api/v1/carrier-events` | ingest an authenticated, idempotent carrier update |
| `GET` | `/api/v1/fulfillments/{id}` | retrieve current fulfillment state |
| `GET` | `/api/v1/fulfillments/{id}/timeline` | retrieve durable tracking history |
| `GET` | `/api/v1/fulfillments/{id}/stream` | stream updates as server-sent events |

The build order and acceptance tests for every route are in the [curriculum map](docs/bookstore-tour/README.md).
