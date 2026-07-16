# 📚 Bookstore API

![Kotlin](https://img.shields.io/badge/Kotlin-2.2%2B-7F52FF?logo=kotlin&logoColor=white)
![Java](https://img.shields.io/badge/Java-21%2B-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.x-6DB33F?logo=springboot&logoColor=white)
![Spring Framework](https://img.shields.io/badge/Spring%20Framework-7-6DB33F?logo=spring&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)
![Architecture](https://img.shields.io/badge/Architecture-Hexagonal-blueviolet)
![Tests](https://img.shields.io/badge/Tests-JUnit5%20%7C%20MockK%20%7C%20Testcontainers%20%7C%20ArchUnit-success)

A small but production-grade REST API for a bookstore's **catalog and order intake**, built with **Kotlin 2.2+ on Spring Boot 4 / Spring Framework 7**, following **hexagonal architecture (ports & adapters)** and SOLID principles.

It is deliberately small in surface (two aggregates: `Book`, `Order`) and deliberately deep in engineering: transactions, optimistic locking, RFC 9457 error contracts, native API versioning, declarative HTTP clients, virtual threads, and a full test pyramid — including tests for the architecture itself.

> 🚌 This repository doubles as a **learning project / Kotlin + Spring Boot 4 refresher**. The full step-by-step walkthrough lives in [`docs/bookstore-guided-tour.md`](docs/bookstore-guided-tour.md), and the design rationale in [`docs/bookstore-api-sdd.md`](docs/bookstore-api-sdd.md).

---

## 🎯 Goal & Scope

### The problem

A bookstore selling through multiple channels (web storefront, in-store terminal, phone orders) has one recurring, expensive failure mode: **selling the last copy of a book twice**. Solving it properly is not a UI problem — it requires:

- **Business invariants enforced in one place** — prices are money (`BigDecimal`, never `Double`), ISBNs are unique, stock never goes negative, and an order's total always equals the sum of its line items at the moment of purchase, no matter which client calls.
- **Correctness under concurrency** — two customers buying the last copy at the same instant must resolve deterministically: exactly one succeeds, the other receives a clear `409 Conflict`.
- **A stable, evolvable contract** — channels come and go; the rules live behind one versioned HTTP API instead of being re-implemented per client.
- **Operability** — every request traceable through the logs, machine-readable errors, health and metrics exposed.

### What this API does

The Bookstore API is the **single system of record** for the catalog and orders:

- **Catalog management** — full CRUD for books with validation, unique-ISBN enforcement, and paginated / sorted / filtered listing (page size capped to prevent pagination-DoS).
- **Order placement** — atomic, transactional order creation: loads books, verifies prices against an external price-check service, checks and decrements stock, computes the total server-side, and persists order + items in a single transaction. Concurrent conflicts on the same book are resolved via optimistic locking (`@Version`) → `409`.
- **Order lifecycle** — `CREATED → PAID | CANCELLED`, persisted as strings (never ordinals).
- **Operational surface** — Actuator health/info/metrics, request-scoped correlation IDs (MDC) in every log line, and RFC 9457 `ProblemDetail` for every non-2xx response.

### Explicit non-goals

Payments, shipping, tax, authentication/authorization, event-driven messaging, microservices, caching-as-correctness, and any frontend. Cutting these keeps the project honest: **small surface, production-grade depth**.

---

## 🛠️ Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Language | **Kotlin 2.2+** (K2 compiler) | JSpecify null-safety interop with Spring APIs |
| Runtime | **Java 21+** | Virtual threads enabled |
| Framework | **Spring Boot 4.x / Spring Framework 7** | Jakarta EE 11 baseline |
| Web | Spring MVC (servlet stack, blocking + virtual threads) | |
| Persistence | Spring Data JPA / Hibernate | JPA confined to the persistence adapter |
| Database | **PostgreSQL 16** | Via Docker Compose, auto-detected by Boot |
| Migrations | **Flyway** | `ddl-auto=validate` — Flyway owns the schema |
| HTTP client | **RestClient** + declarative HTTP interfaces (`@HttpExchange`) | No Retrofit/RestTemplate/WebClient |
| Build | Gradle (Kotlin DSL) | `plugin.spring` (all-open) + `plugin.jpa` (no-arg) |
| Testing | **JUnit 5, MockK, Testcontainers, ArchUnit** | Real Postgres in integration tests |

---

## 🌱 Spring features exercised in this project

**Core & configuration**
- `@SpringBootApplication`, profiles, `@ConfigurationPropertiesScan`
- Constructor-bound, immutable `@ConfigurationProperties` data classes (typed config over `@Value`)
- Docker Compose integration (`spring-boot-docker-compose`)

**Web / API**
- `@RestController`, request mappings, `@PathVariable`, `@RequestBody`, `ResponseEntity` (201 + `Location`, 204, 404, 409)
- Bean Validation (`@Valid`, jakarta annotations with Kotlin `@field:` use-site targets)
- **Native API versioning (Spring 7)** — `@GetMapping(version = "1.1")` with a configured `ApiVersionStrategy`, RFC 9745 deprecation signaling
- Pagination & sorting with `Pageable` / `Page`, derived queries and JPQL `@Query`
- Global error handling: `@RestControllerAdvice` + **RFC 9457 `ProblemDetail`**

**Persistence & transactions**
- JPA entity mapping (`@Entity`, `@Version`, `@Enumerated(STRING)`, associations), JPA auditing (`@CreatedDate` / `@LastModifiedDate`)
- Spring Data repositories with Kotlin nullable returns (no `Optional`)
- `@Transactional` order placement with full rollback semantics
- **Optimistic locking** — concurrent last-copy purchases resolve to one commit + one `409`
- Flyway migrations with `ddl-auto=validate`

**Outbound HTTP & resilience (Spring 7 / Boot 4)**
- Declarative **HTTP interface client** (`@GetExchange`) registered with **`@ImportHttpServices`**, backed by `RestClient`
- In-framework resilience: **`@Retryable`** (backoff + jitter) and **`@ConcurrencyLimit`** on the external price-check adapter

**Concurrency**
- **Virtual threads** (`spring.threads.virtual.enabled=true`) as the default execution model
- One comparative implementation using **Kotlin coroutines** (structured concurrency fan-out for external calls) — see the design docs for the trade-off discussion

**Observability & operations**
- Actuator (`health`, `info`, `metrics` only)
- MDC request-correlation filter (`requestId` in every log line), SLF4J with parameterized messages

**Testing**
- Plain JUnit for the framework-free domain
- MockK unit tests against ports
- `@WebMvcTest` slice tests (status codes, JSON shape, validation 400s)
- `@DataJpaTest` / `@SpringBootTest` with **Testcontainers** Postgres via `@ServiceConnection`
- **ArchUnit** tests enforcing the hexagonal dependency rule
- `RestTestClient` (new in Spring 7)

---

## 🏛️ Architecture

Hexagonal (ports & adapters), enforced by ArchUnit rather than convention. **The one rule: dependencies point inward.** The domain has zero Spring/JPA imports and is testable in milliseconds without a container.

```
com.example.bookstore
├── domain/                  # pure Kotlin — business model, rules, errors (zero framework imports)
├── application/
│   ├── port/in/             # use cases: PlaceOrderUseCase, ManageBooksUseCase, ...
│   ├── port/out/            # contracts the core needs: BookRepositoryPort, PriceCheckPort, ...
│   └── service/             # use-case implementations (@Transactional lives here)
├── adapter/
│   ├── in/web/              # REST controllers, DTOs, ProblemDetail advice
│   └── out/
│       ├── persistence/     # JPA entities + Spring Data repos + port adapters
│       └── pricing/         # HTTP interface client + port adapter (@Retryable)
└── infrastructure/          # main class, config, MDC logging filter
```

Key decisions (full rationale in [`docs/bookstore-api-sdd.md`](docs/bookstore-api-sdd.md)):
- Domain model **separate** from JPA entities; mapping via Kotlin extension functions.
- Interfaces **only at architectural boundaries** (ports) — no `ServiceImpl` ceremony.
- Blocking MVC + **virtual threads** by default; coroutines where structured concurrency earns its keep.
- **RestClient + HTTP interfaces** over Retrofit/RestTemplate/WebClient for outbound calls.

---

## 🚀 Getting started

**Prerequisites:** JDK 21+, Docker (for Compose & Testcontainers).

```bash
# clone & run — Boot starts Postgres via compose.yaml and Flyway migrates automatically
./gradlew bootRun

# run the full test suite (unit, slice, Testcontainers integration, ArchUnit)
./gradlew test
```

The API is served at `http://localhost:8080`.

### API overview

| Method | Path | Description | Status codes |
|---|---|---|---|
| `POST` | `/api/v1/books` | Create a book | `201` + `Location`, `400`, `409` (duplicate ISBN) |
| `GET` | `/api/v1/books/{id}` | Get a book (versioned: `1.0` / `1.1`) | `200`, `404` |
| `PUT` | `/api/v1/books/{id}` | Update a book | `200`, `400`, `404` |
| `DELETE` | `/api/v1/books/{id}` | Delete a book | `204`, `404` |
| `GET` | `/api/v1/books` | List: `?page=&size=&sort=&author=&maxPrice=` | `200` |
| `POST` | `/api/v1/orders` | Place an order | `201`, `400` (limit/validation), `409` (stock/conflict) |
| `GET` | `/api/v1/orders/{id}` | Get an order with items | `200`, `404` |

All errors follow **RFC 9457**:

```json
{
  "type": "https://example.com/problems/insufficient-stock",
  "title": "Insufficient stock",
  "status": 409,
  "detail": "Book 42 has 1 unit(s) left, 2 requested"
}
```

---

## 📖 Documentation

- [`docs/bookstore-guided-tour.md`](docs/bookstore-guided-tour.md) — the step-by-step **guided learning tour** (build order, traps, interview talking points)
- [`docs/bookstore-api-sdd.md`](docs/bookstore-api-sdd.md) — the **software design document** (scope, ADRs, trade-offs)

## 📄 License

MIT — see [LICENSE](LICENSE).
