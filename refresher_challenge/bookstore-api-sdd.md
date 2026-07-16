# 📚 Bookstore API — Software Design Document
### Kotlin 2.2+ · Java 17/21+ · Spring Framework 7 · Spring Boot 4 · Hexagonal Architecture

> **Document type:** Scope-first SDD for a deliberately small, feature-dense backend.
> **Dual purpose:** (1) a real, well-architected service specification and (2) a structured Kotlin + Spring Boot 4 refresher — every architectural decision doubles as an interview talking point.

---

# PART I — SCOPE

## 1. What the software is

The **Bookstore API** is the backend system of record for a small bookstore's **catalog** and **order intake**. It is a stateless HTTP/JSON service that:

- maintains the inventory of books (title, author, ISBN, price, stock level),
- accepts customer orders against that inventory,
- guarantees that stock is never oversold, even under concurrent purchases,
- exposes its state to operators (health, metrics) and to consuming clients (paged, filtered catalog queries),
- consults an external price-verification service before accepting an order (simulated third party).

It is the *authoritative source of truth*: any storefront, mobile app, or internal admin tool is a client of this API. Nothing else writes to the bookstore database.

## 2. Problem context — why this system exists

A bookstore that sells through multiple channels (web storefront, in-store terminal, phone orders) has one recurring, expensive failure mode: **selling the same last copy twice**. The naive fix — a single shared spreadsheet or a UI talking straight to a database — breaks down because:

1. **Concurrency.** Two customers buying the last copy at the same instant must resolve deterministically: exactly one succeeds, the other gets a clear, actionable error. This is a *transactional consistency* problem, not a UI problem.
2. **Integrity.** Prices are money (`BigDecimal`, never floating point), ISBNs are unique identities, stock can never go negative, and an order's total must equal the sum of its line items *at the moment of purchase* — these are **business invariants** that must hold no matter which client calls in.
3. **Evolution.** Channels come and go (a mobile app today, a partner marketplace tomorrow). The rules of the bookstore must not be re-implemented in each channel; they must live in one place, behind a stable contract that can be **versioned** without breaking existing clients.
4. **Operations.** When an order fails at 2 a.m., someone must be able to trace *that one request* across the logs, know whether the database is healthy, and see error responses that machines can parse (RFC 9457) instead of raw stack traces.

The Bookstore API solves exactly this problem class: **centralize the business rules and the consistency guarantees behind one versioned HTTP contract.**

### 2.1 The second problem this project solves (the meta-scope)

This codebase is *also* a **learning vehicle**: a Kotlin refresher targeting Spring Boot 4 / Spring Framework 7 APIs. The domain was chosen to be the smallest one that still forces contact with the features that dominate real API work and interviews:

| Real-world concern | Feature it forces you to exercise |
|---|---|
| Two aggregates with a relation | JPA mappings, `@OneToMany`, fetch strategies, N+1 |
| Money + identity invariants | `BigDecimal`, validation, unique constraints |
| "Last copy" races | `@Transactional`, `@Version` optimistic locking |
| Multiple clients, long-lived contract | DTOs, API versioning (native in Spring 7), ProblemDetail |
| An external dependency | HTTP interface clients, `@Retryable`, resilience |
| Someone has to run it | Actuator, MDC/correlation IDs, typed configuration |
| Someone has to change it safely | Hexagonal boundaries, MockK, Testcontainers, ArchUnit |

Buildable in a handful of evenings; two aggregates (`Book`, `Order`) are enough for relations, transactions and business rules without drowning in domain complexity.

## 3. Functional scope (what the software does)

### 3.1 Catalog management
- Create a book (ISBN must be unique; price positive; stock ≥ 0) → `201 Created` + `Location`.
- Read a book by id → `200` / `404`.
- Update a book → `200` / `404`.
- Delete a book → `204` / `404`.
- List books: paginated, sortable, filterable by author (contains, case-insensitive) and max price; page size capped (default 20, max 100) to prevent pagination-DoS.

### 3.2 Order placement
- `POST /api/v1/orders` with `{ customerEmail, items: [{bookId, quantity}] }`:
  - validates the request shape (email format, positive quantities),
  - enforces **max items per order** (externalized, typed configuration),
  - loads each book, verifies price against the external price-check service,
  - checks and **atomically decrements stock** for every line item,
  - computes the total server-side (client-supplied totals are never trusted),
  - persists order + items in **one transaction** — any failure rolls back everything,
  - resolves concurrent conflicts on the same book via optimistic locking → `409 Conflict`.
- `GET /api/v1/orders/{id}` returns the order with its items (no lazy-loading leaks).
- Order lifecycle: `CREATED → PAID | CANCELLED` (stored as `STRING`, never `ORDINAL`).

### 3.3 Operational surface
- Health, info and metrics via Actuator (only those three exposed).
- Every log line carries a request-scoped correlation id (`requestId` in MDC).
- Machine-readable errors: every non-2xx response is an RFC 9457 `ProblemDetail`.
- One endpoint demonstrates Spring 7 **native API versioning** (`version = "1.1"` mapping) so the evolution story is proven, not theoretical.

### 3.4 Explicit non-goals (out of scope)
- Payments, shipping, tax, multi-currency — `PAID` is a status flip, not a gateway integration.
- Authentication/authorization (Spring Security is its own refresher project; the architecture leaves a clean seam for it in the web adapter).
- Event-driven anything (outbox, Kafka), microservices, CQRS, caching-as-correctness.
- Frontend, admin UI, reporting.

Cutting these is what keeps the project honest: **small surface, production-grade depth.**

## 4. Actors and external systems

| Actor | Interaction |
|---|---|
| Catalog manager (via admin client) | CRUD on books |
| Storefront / ordering client | Browses catalog, places orders |
| Operations / SRE | Actuator endpoints, structured logs |
| **External price-check service** (simulated) | Outbound HTTP call during order placement — exists to force an outbound-adapter + resilience story |
| PostgreSQL 16 | Owned exclusively by this service; schema evolved only via Flyway |

## 5. Quality attributes (the "-ilities" that drove the architecture)

1. **Consistency under concurrency** — the defining requirement. Optimistic locking + single-transaction order placement.
2. **Evolvability** — business rules isolated from frameworks; new delivery mechanisms or storage engines are new adapters, not rewrites. Contract versioned.
3. **Testability** — domain logic testable with plain JUnit (no Spring context, milliseconds); each adapter testable in isolation; the whole thing verifiable against a *real* Postgres via Testcontainers.
4. **Observability** — correlation ids, standard error contract, health/metrics.
5. **Comprehensibility** — a new developer (or you, in six months) can find any rule by asking "is this business or plumbing?" and opening exactly one package.

---

# PART II — ARCHITECTURE

## 6. Architectural style: Hexagonal (Ports & Adapters), package-enforced

The system follows **hexagonal architecture** (Alistair Cockburn, 2005), which is Clean Architecture's most practical incarnation for a Spring service: the **domain core** sits in the middle knowing nothing about HTTP, JPA or Spring; everything technical is an **adapter** plugged into a **port** (a Kotlin interface owned by the core). Dependencies point strictly inward.

```
            ┌──────────────────────────────────────────────────┐
  driving   │                    adapters                      │  driven
  (input)   │  ┌────────────────────────────────────────────┐  │  (output)
            │  │              application                   │  │
 REST ──────┼─▶│  input ports   ┌──────────┐  output ports  │◀─┼────── JPA/Postgres
 controller │  │  (use cases)   │  domain  │  (interfaces)  │  │
            │  │                │ Book,    │                │◀─┼────── RestClient →
 (future:   │  │  PlaceOrder    │ Order,   │ BookRepository │  │       price service
  GraphQL,  │  │  ManageBooks   │ rules    │ PriceGateway   │  │
  CLI, MQ)  │  └────────────────└──────────┘────────────────┘  │
            └──────────────────────────────────────────────────┘
                        dependency direction: always inward →◀←
```

**The dependency rule, concretely:**

| Layer | May depend on | Must NOT depend on |
|---|---|---|
| `domain` | Kotlin stdlib, `java.time`, `java.math` | Spring, JPA, Jackson, anything in `adapter.*` |
| `application` | `domain` (+ minimal Spring for `@Transactional` — see ADR-006) | web, persistence, HTTP client classes |
| `adapter.in.web` | `application` ports, `domain` types it maps | `adapter.out.*` |
| `adapter.out.persistence` / `adapter.out.pricing` | the output-port interfaces + `domain` | `adapter.in.*` |

### 6.1 Package layout (single Gradle module — see ADR-002)

```
com.example.bookstore
├── domain/                        # pure Kotlin — zero framework imports
│   ├── model/          Book.kt, Order.kt, OrderItem.kt, OrderStatus.kt, Isbn.kt
│   └── error/          BookNotFoundException.kt, InsufficientStockException.kt, ...
├── application/
│   ├── port/
│   │   ├── in/         PlaceOrderUseCase.kt, ManageBooksUseCase.kt, QueryBooksUseCase.kt
│   │   └── out/        BookRepositoryPort.kt, OrderRepositoryPort.kt, PriceCheckPort.kt
│   └── service/        PlaceOrderService.kt, BookService.kt          # implement input ports
├── adapter/
│   ├── in/web/         BookController.kt, OrderController.kt, dto/, GlobalExceptionHandler.kt
│   └── out/
│       ├── persistence/ BookJpaEntity.kt, SpringDataBookRepository.kt, BookPersistenceAdapter.kt
│       └── pricing/     PriceCheckClient.kt (HTTP interface), PriceCheckAdapter.kt
└── infrastructure/     BookstoreApplication.kt, config/, logging/ (MDC filter), OrderProperties.kt
```

Boundaries are **enforced by ArchUnit tests** (M9), not by convention and good intentions.

---

## 7. Architecture Decision Records

Each ADR: context → decision → why → what you learn defending it.

### ADR-001 · Hexagonal over classic three-layer — with eyes open

**Context.** The default Spring tutorial layout (`controller → service → repository`) works, but the "service" layer inevitably accumulates HTTP concerns, JPA concerns and business rules in one place, and the domain ends up shaped like the database.

**Decision.** Ports & adapters, with a **framework-free domain**: the `domain` package compiles with zero Spring/JPA imports, and business rules are testable with plain JUnit in milliseconds.

**Honest cost accounting** (this is the part interviewers respect): hexagonal buys you replaceable technology, isolated tests and deferred decisions — and charges you **mapping boilerplate** (JPA entity ↔ domain model), more files, and team discipline. For a trivial CRUD app it is overkill; the practitioner consensus is that it pays off when there is real business logic, multiple interfaces, or long-term maintenance — which is precisely what we're simulating, and why it's worth learning on a small codebase where the pattern stays visible.

**🎤 Talking point:** "Hexagonal isn't about the folder names. It's one rule — *dependencies point inward* — plus interfaces at the boundaries so I can compile and test the business core without booting a framework."

### ADR-002 · Single Gradle module + ArchUnit, not multi-module

**Context.** Strict implementations split `core` / `api` / `data` into separate Gradle modules so the compiler physically prevents illegal imports. Multi-module is the stronger guarantee but adds build ceremony that distracts from the learning goals.

**Decision.** One module; boundaries enforced by **ArchUnit** rules (`noClasses().that().resideInAPackage("..domain..").should().dependOnClassesThat().resideInAPackage("org.springframework..")` and layer checks). If this grew into a team codebase, promoting packages to modules is a mechanical refactor precisely *because* the dependencies already point the right way.

### ADR-003 · Interfaces between layers: only at architectural boundaries

**Context.** You asked whether to put interfaces between layers. There are two very different practices hiding under that phrase:

1. **Ports** — interfaces that mark an *architectural boundary* (core ↔ outside world).
2. **The `FooService` / `FooServiceImpl` reflex** — an interface for every class, same package, single implementation.

**Decision.** (1) always; (2) never.

- **Output ports are mandatory.** `BookRepositoryPort` and `PriceCheckPort` are Kotlin interfaces *owned by the application layer*; the JPA adapter and the HTTP adapter implement them. This is the **Dependency Inversion Principle** doing real work: the core defines the contract, infrastructure conforms to it, and in tests you substitute a fake in one line.
- **Input ports (use-case interfaces) are included here for the learning value** — they make the "what does this system do?" list greppable (`PlaceOrderUseCase`) and keep controllers ignorant of implementation classes. Know the counter-argument: many production teams skip them and let controllers call application services directly, because controller → service already points inward. Skipping input ports is a defensible pragmatic trim; skipping output ports collapses the architecture.
- **`ServiceImpl` pairs add nothing**: Spring has proxied concrete classes via CGLIB for a decade (that's what `plugin.spring`/all-open is for), MockK mocks final classes, and a single-implementation interface in the same package is speculative generality. Add an interface *within* a layer only when a second implementation or a boundary actually exists.

**🎤 Talking point:** "DIP is about the *direction of the dependency arrow*, not about the interface keyword. I use interfaces where the arrow needs to flip — at ports — and concrete classes everywhere else."

### ADR-004 · SOLID, mapped to actual code decisions

| Principle | Where it materializes in this codebase |
|---|---|
| **S**RP | Controllers translate HTTP only; use-case services orchestrate one business operation each; adapters translate technology. A change in JSON shape, business rule, or SQL touches exactly one place. |
| **O**CP | Adding a GraphQL API or swapping Postgres for Mongo = new adapter implementing an existing port; zero edits to `domain`/`application`. |
| **L**SP | Every `BookRepositoryPort` implementation (JPA adapter, in-memory test fake) must honor the same contract — e.g., `findByIsbn` returns `null` (not throws) when absent. Test fakes that cheat the contract are LSP violations that poison your tests. |
| **I**SP | Ports are role-shaped and small (`PriceCheckPort` has one method), so no adapter is forced to stub methods it doesn't serve. |
| **D**IP | High-level policy (`PlaceOrderService`) depends on abstractions it owns (`out` ports); low-level detail (JPA, RestClient) depends on those abstractions. Constructor injection of `val` port references makes the graph explicit and immutable. |

### ADR-005 · Domain model separate from JPA entities

**Context.** The single sharpest tension in "hexagonal + Spring Data JPA": JPA's whole value proposition is *one* annotated model, while hexagonal wants persistence-ignorant domain objects. Respected practitioners note that keeping JPA annotations on your domain objects quietly undermines the architecture — the ORM starts dictating your design (mutable `var`s, nullable ids, no-arg constructors, open classes).

**Decision.** Separate them. `domain.model.Book` is a proper Kotlin class enforcing its own invariants (`init { require(price > BigDecimal.ZERO) }`); `adapter.out.persistence.BookJpaEntity` is the JPA-shaped mutable class; the persistence adapter maps between them with extension functions (`fun BookJpaEntity.toDomain(): Book`).

**Consequence accepted:** mapping boilerplate for two aggregates. That's the tuition for this lesson — and you'll be able to articulate *exactly* why teams sometimes take the pragmatic shortcut (entities-as-domain) and what they give up.

### ADR-006 · `@Transactional` lives on application services (a documented leak)

**Context.** Order placement must be atomic. But `@Transactional` is a Spring annotation, and the application layer is supposed to be framework-light. Purists wrap transactions behind a port or decorator; pragmatists annotate the use-case service.

**Decision.** Annotate `PlaceOrderService.place()` with `@Transactional` and record it as a **conscious, contained exception** to framework-freedom: it's declarative metadata, not control flow, and the alternative machinery costs more than it teaches. The *domain* stays pure; the application layer accepts two Spring imports (`@Transactional`, stereotype for scanning).

**🎤 Talking points (unchanged interview gold):**
- Proxy-based ⇒ **self-invocation bypasses the transaction** (`this.otherTxMethod()` never hits the proxy).
- Default rollback: unchecked only. Kotlin has no checked exceptions, but Spring still applies the Java rule *by exception type* — a Kotlin function throwing `IOException` will NOT roll back by default. Know `rollbackFor`.
- `readOnly = true` on queries: skips dirty checking, can hint replica routing.

### ADR-007 · Concurrency: blocking MVC + virtual threads by default; coroutines where they earn it

**Context (the question you actually asked).** Kotlin coroutines vs Java 21+ virtual threads is *the* 2026 concurrency interview topic, and this app has to pick.

**Decision.** **Spring MVC, blocking style, with `spring.threads.virtual.enabled=true`.** One stretch milestone (M10) converts the outbound price-check orchestration to `suspend` functions to feel the difference.

**Why virtual threads are the default here:**
- The hot path is **JPA/JDBC — a blocking API**. Coroutines don't make JDBC non-blocking; they'd just shuffle the blocking onto another thread pool. Virtual threads make the blocking *cheap* with **zero code changes** — the thread-per-request model simply stops costing an OS thread per request.
- Boot enables it with one property; every `RestClient` call, every repository call scales without `suspend` coloring your entire call stack.
- Spring 7's new `@ConcurrencyLimit` (resilience moved into the framework, alongside `@Retryable`) is designed with virtual threads in mind — unbounded cheap threads still need bulkheads around scarce resources like DB connection pools.

**Where coroutines win — and when you'd flip the decision:**
- **Structured concurrency**: fanning out several remote calls with automatic cancellation of siblings on failure (`coroutineScope { async{} async{} }`) is where coroutines are genuinely superior; virtual threads need explicit `StructuredTaskScope` for comparable lifecycle guarantees.
- A **truly reactive stack** (WebFlux + R2DBC): Kotlin coroutines are the idiomatic face of it (`Flow`, suspending repositories) — far more readable than raw Reactor.
- The honest 2026 summary: for a typical blocking MVC + JPA service the throughput difference is academic; virtual threads = scale without rewriting, coroutines = structured concurrency and cancellation when your code composes many async steps. Also note they compose: coroutines can dispatch onto virtual threads (`Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()`).

**🎤 Talking point:** "Suspending is a property of the *code*; virtual threads are a property of the *runtime*. My JPA app blocks either way, so I let the runtime absorb it. I'd reach for coroutines the moment I'm composing multiple remote calls and need structured cancellation, or if I were on R2DBC."

### ADR-008 · HTTP client: `RestClient` + declarative HTTP interfaces — not Retrofit

**Context (your other question).** For the outbound price-check call the candidates are Retrofit, `RestTemplate`, `WebClient`, `RestClient`, and Spring 6/7's declarative HTTP interfaces.

**Decision.** A declarative **HTTP interface** (`@GetExchange` on a Kotlin interface) backed by **`RestClient`**, registered with Boot 4's `@ImportHttpServices` — which reduces the old manual `HttpServiceProxyFactory` wiring to a single annotation.

**The elimination round:**
- **`RestTemplate`** — maintenance mode; feature-frozen. Legacy code only.
- **`WebClient`** — brings the entire reactive stack (Reactor, `Mono`/`Flux`) into a blocking servlet app for no benefit. Only justified on WebFlux.
- **Retrofit** — a genuinely good library, and the *inspiration* for Spring's HTTP interfaces (both are "annotated interface → generated client"). But in a Spring Boot 4 service it's a guest: you hand-wire OkHttp, converters and factories, add a dependency surface, and forfeit native integration. Spring's own HTTP interfaces give you the identical declarative ergonomics **natively**, participate in Boot auto-configuration and test support, ride on `RestClient` (so they're virtual-thread-friendly and reactive-free), and in Spring 7 even support **API versioning at the client** via `ApiVersionInserter` and per-group configuration (`HttpServiceGroupConfigurer`). Retrofit remains the right answer on Android; on the server, the framework caught up.
- Bonus: Spring 7 lets you slap **`@Retryable`** (now in-framework, with backoff + jitter) and `@ConcurrencyLimit` on the adapter method — resilience without importing resilience4j for simple cases.

```kotlin
interface PriceCheckClient {                      // adapter.out.pricing
    @GetExchange("/prices/{isbn}")
    fun currentPrice(@PathVariable isbn: String): PriceQuote
}

@Configuration
@ImportHttpServices(PriceCheckClient::class)      // Boot 4: that's the whole wiring
class HttpClientsConfig
```

The application core never sees this interface — it sees `PriceCheckPort`; `PriceCheckAdapter` implements the port and delegates to the generated client. Swap Retrofit in later and *nothing* inside the hexagon changes — which is the architecture proving its worth.

### ADR-009 · API contract: DTOs, ProblemDetail, capped paging, native versioning

- **DTO boundary is absolute.** Entities/domain objects never serialize to JSON. Request/response types are Kotlin `data class`es (here they're *ideal* — value semantics, `copy()` for test builders), mapped via extension functions.
- **Errors** are RFC 9457 `ProblemDetail` from one `@RestControllerAdvice` — a standard, machine-readable contract (`type`, `title`, `status`, `detail` + a field-error map for validation failures). Mention that 9457 supersedes 7807 and you sound current.
- **Paging** accepts `Pageable`, returns mapped `Page<BookResponse>`, caps size at 100 (pagination-DoS). Hexagonal nuance worth knowing: passing Spring's `Pageable` through the input port leaks framework into the core — the strict fix is your own small `PageQuery` type; teams often accept the leak for query-only paths. We define `PageQuery` (it's ~5 lines) and say why.
- **Versioning**: URL prefix `/api/v1` as the coarse contract, *plus* one endpoint demonstrating Spring 7's first-class versioning — `@GetMapping(version = "1.1")` with an `ApiVersionConfigurer` strategy (path/header/query/media-type all supported, with RFC 9745 deprecation signaling). This feature is new headline material in Framework 7; showing it in a portfolio project is cheap and differentiating.

### ADR-010 · Persistence, schema and Kotlin/JPA discipline

- **Flyway owns the schema**; `spring.jpa.hibernate.ddl-auto=validate`. "I never let Hibernate generate schema in prod" remains a free interview point.
- Money = `NUMERIC(10,2)` ↔ `BigDecimal`. Never `Double`.
- JPA entities are **normal classes with `var`s, never `data class`es** — generated `equals`/`hashCode` over mutable + DB-generated fields breaks identity across the persistence lifecycle, `toString` can detonate lazy associations, `copy()` invites detached duplicates. Data classes: DTOs yes, entities no.
- `@Version` on the book entity ⇒ optimistic locking ⇒ concurrent last-copy purchase resolves as `ObjectOptimisticLockingFailureException` → mapped to `409`.
- Kotlin build plugins are load-bearing, not boilerplate: `plugin.spring` (all-open, because Spring proxies by subclassing and Kotlin is final-by-default) and `plugin.jpa` (no-arg constructors for Hibernate's reflective instantiation).
- **Null-safety at the boundary:** Boot 4 / Framework 7 completed the migration to **JSpecify** annotations, and Kotlin 2.1+ translates them into real Kotlin nullability — Spring API returns are `User?`, not platform types. Keep `-Xjsr305=strict` for remaining third-party libs. This is *the* Kotlin-specific "why Boot 4 matters" line.

### ADR-011 · Testing strategy = the architecture, verified

| Ring | Tooling | What it proves |
|---|---|---|
| Domain | Plain JUnit 5 — **no Spring, no mocks** | Business invariants; runs in ms |
| Application | MockK fakes/mocks of *ports* | Orchestration + business exceptions; `every`/`verify`/`slot` |
| Web adapter | `@WebMvcTest` + mocked input port | Status codes, JSON shape, validation 400s, ProblemDetail shape |
| Persistence adapter | `@DataJpaTest` + **Testcontainers Postgres** via `@ServiceConnection` | Real SQL, real constraints (H2 lies) |
| Whole hexagon | `@SpringBootTest` + Testcontainers | Place-order flow incl. rollback & optimistic-lock 409 |
| Architecture | **ArchUnit** | The dependency rule itself |

MockK over Mockito stays the Kotlin answer: final classes, `suspend` functions, extension functions, Kotlin DSL.

---

# PART III — BUILD PLAN (the learning curve, in order)

Same strategy as before — milestone-driven, each one refreshing a named cluster of features — but the sequence now *follows the architecture inward-out*: domain first (pure Kotlin), then ports, then adapters. You experience the payoff of hexagonal instead of reading about it: **M1 and M2 compile and pass tests before Spring Web or JPA exist in the project.**

## Tech stack (current as of mid-2026)

| Piece | Choice | Why |
|---|---|---|
| Spring Boot | **4.x** (Framework 7) | Baselines: Kotlin 2.2, Java 17+ (happily runs 21/25 — use **21+** to get virtual threads), Jakarta EE 11; Boot 3.5 hit OSS EOL June 2026 |
| Kotlin | 2.2+ | K2 compiler; JSpecify interop |
| Build | Gradle Kotlin DSL | Type-safe, the Kotlin-shop default |
| DB | PostgreSQL 16 (Docker Compose, auto-detected via `spring-boot-docker-compose`) | |
| Migrations | Flyway | `ddl-auto=validate` |
| Testing | JUnit 5 + MockK + Testcontainers + ArchUnit | |

Bootstrap at start.spring.io → Kotlin, Gradle-Kotlin → **Web, Data JPA, Validation, PostgreSQL, Flyway, Actuator, Testcontainers, Docker Compose support**.

```kotlin
plugins {
    id("org.springframework.boot") version "4.0.x"
    id("io.spring.dependency-management") version "1.1.x"
    kotlin("jvm") version "2.2.x"
    kotlin("plugin.spring") version "2.2.x"  // ⭐ all-open for Spring proxies
    kotlin("plugin.jpa") version "2.2.x"     // ⭐ no-arg ctor for Hibernate
}
kotlin { compilerOptions { freeCompilerArgs.add("-Xjsr305=strict") } }
```

```yaml
# compose.yaml
services:
  postgres:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: bookstore, POSTGRES_USER: bookstore, POSTGRES_PASSWORD: secret }
    ports: ["5432:5432"]
```

## Database schema (Flyway `V1__create_books.sql`, `V2__create_orders.sql`)

```
books:        id (bigserial PK), isbn (unique, not null), title, author,
              price NUMERIC(10,2), stock int, version bigint, created_at, updated_at
orders:       id, customer_email, status (CREATED|PAID|CANCELLED), total, created_at
order_items:  id, order_id (FK), book_id (FK), quantity, unit_price
```

---

### ✅ M0 — Walking skeleton + typed configuration
**Refreshes:** `@SpringBootApplication`, profiles, `@ConfigurationProperties`, Docker Compose integration.

1. App boots against Postgres; Flyway runs V1; `ddl-auto=validate`.
2. Bind config to an immutable class and enable with `@ConfigurationPropertiesScan`:
```kotlin
@ConfigurationProperties(prefix = "bookstore.orders")
data class OrderProperties(val maxItemsPerOrder: Int = 10)
```
3. Add `application-local.yml`; run with the profile.

**🎤** Constructor-bound `@ConfigurationProperties` data classes > `@Value`: type-safe, immutable, validatable (`@Validated` + jakarta annotations), metadata-discoverable.

### ✅ M1 — Domain core (pure Kotlin, zero Spring)
**Refreshes:** Kotlin design muscle — `data class` vs class, sealed hierarchies, `init` invariants, `require`, value semantics.

1. Write `domain.model`: `Book` (enforces positive price, non-negative stock, exposes `decreaseStock(qty)` that throws `InsufficientStockException`), `Order` + `OrderItem` (total computed from items), `OrderStatus` enum.
2. Business exceptions in `domain.error`.
3. **Plain JUnit tests** — no Spring context, no mocks. `Order` math, stock rules, invariant violations.

**The point:** this package would compile in a Ktor app, a CLI, or a Lambda unchanged. That's the hexagon's core, demonstrated.

### ✅ M2 — Ports + application services
**Refreshes:** interfaces-as-contracts, constructor injection, DIP.

1. Define `port.in` (use cases) and `port.out` (`BookRepositoryPort`, `OrderRepositoryPort`, `PriceCheckPort`) as Kotlin interfaces with **nullable returns instead of `Optional`** (`fun findByIsbn(isbn: String): Book?`).
2. Implement `BookService`, `PlaceOrderService` against the ports. Enforce `maxItemsPerOrder` from M0's properties here.
3. Unit test with MockK against the ports — the last-copy scenario, duplicate ISBN, item-limit breach — still no Spring context.

**🎤** Constructor injection over field injection: immutability (`val`), can't exist in an invalid state, trivially testable, circular deps fail fast — and in Kotlin, field injection would force `lateinit var`, surrendering null-safety.

### ✅ M3 — Persistence adapter (the Kotlin/JPA minefield)
**Refreshes:** `@Entity`, `@Version`, auditing, Spring Data repositories, entity↔domain mapping.

1. `BookJpaEntity` as a **normal class** (see ADR-010 for the full data-class indictment):
```kotlin
@Entity @Table(name = "books")
class BookJpaEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) val id: Long? = null,
    @Column(nullable = false, unique = true) var isbn: String,
    var title: String, var author: String,
    var price: BigDecimal, var stock: Int,
    @Version var version: Long = 0,
)
```
2. Auditing: `@CreatedDate`/`@LastModifiedDate` + `@EntityListeners(AuditingEntityListener::class)` + `@EnableJpaAuditing`.
3. `SpringDataBookRepository : JpaRepository<BookJpaEntity, Long>` stays `internal` to the adapter package; `BookPersistenceAdapter` implements `BookRepositoryPort` and maps with `fun BookJpaEntity.toDomain(): Book` / `fun Book.toEntity(): BookJpaEntity`.
4. `@DataJpaTest` + Testcontainers `@ServiceConnection` — one annotation, real Postgres, no H2 lies.

### ✅ M4 — Web adapter: CRUD + DTOs + validation
**Refreshes:** `@RestController`, mapping annotations, `@Valid`, `ResponseEntity`, the `@field:` trap.

Endpoints per §3.1; DTOs as data classes; mapping via extension functions.

```kotlin
data class CreateBookRequest(
    @field:NotBlank @field:Size(min = 10, max = 17) val isbn: String,
    @field:NotBlank val title: String,
    @field:NotBlank val author: String,
    @field:Positive val price: BigDecimal,
    @field:PositiveOrZero val stock: Int,
)
```

**⚠️ The trap everyone hits:** without `@field:`, Kotlin sticks the annotation on the constructor *parameter* where Bean Validation may never look — validation silently does nothing. Boot 4 improves defaults, but *why `@field:` exists* is the interview answer.

### ✅ M5 — Centralized errors with ProblemDetail
**Refreshes:** `@RestControllerAdvice`, `@ExceptionHandler`, RFC 9457.

Map: `BookNotFoundException`→404 · `DuplicateIsbnException`→409 · `MethodArgumentNotValidException`→400 with field-error map in `properties` · `ObjectOptimisticLockingFailureException`→409 · fallback `Exception`→500 generic (never leak stack traces). Verify the JSON shape (`type`, `title`, `status`, `detail`) in tests.

### ✅ M6 — Logging done properly
**Refreshes:** SLF4J idioms in Kotlin, levels, MDC.

1. Pick and defend a logger idiom: companion-object `LoggerFactory.getLogger(...)` or a reified top-level helper `inline fun <reified T> T.logger(): Logger`.
2. Parameterized messages — `log.info("Book created id={}", id)` — never `"$id"` interpolation (evaluated even when the level is off).
3. `OncePerRequestFilter` that seeds **MDC** with a `requestId`; `%X{requestId}` in the pattern. That's the "trace one request across logs" answer.
4. `logging.level.org.hibernate.SQL=debug` in the local profile; know why `show-sql=true` is worse (bypasses the log system).

### ✅ M7 — Pagination, sorting, filtering
**Refreshes:** `Pageable`, `Page<T>`, derived queries, `@Query`.

1. `GET /api/v1/books?page=0&size=20&sort=title,asc` — controller takes `Pageable`, maps `page.map { it.toResponse() }`, cap size at 100.
2. Filters: one derived query (`findByAuthorContainingIgnoreCase`) + one JPQL `@Query` with `@Param` — both styles refreshed.
3. Hexagonal wrinkle: the input port speaks a tiny `PageQuery`/`PagedResult` pair, the adapter translates to Spring's `Pageable`. Feel the cost, understand the shortcut (ADR-009).

### ✅ M8 — Orders: transactions + optimistic locking (the meaty one)
**Refreshes:** `@Transactional`, propagation, `@Version` conflicts, `@Enumerated(STRING)`, associations, lazy-loading.

1. `PlaceOrderService.place()` in **one transaction**: load books → price-check via `PriceCheckPort` → validate stock → decrement → compute total → persist order + items. Any failure rolls back all of it.
2. Business rules → business exceptions → advice: `InsufficientStockException`→409, item limit→400.
3. Two-thread test proving the last-copy race yields exactly one success and one 409.
4. `GET /orders/{id}`: dodge `LazyInitializationException` — map to DTO *inside* the transaction or fetch-join.
5. All ADR-006 talking points apply (self-invocation, rollback rules, `readOnly`).

### ✅ M9 — Tests completed + ArchUnit
Fill the pyramid per ADR-011 and add the architecture tests:

```kotlin
@AnalyzeClasses(packages = ["com.example.bookstore"])
class HexagonalRulesTest {
    @ArchTest val domainIsPure = noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "org.springframework..", "jakarta.persistence..", "..adapter..")
}
```

### ✅ M10 — Spring 7 / Boot 4 feature tour (the "most-used current API features" pass)
**Refreshes:** what's actually new — the differentiators for 2026 interviews.

1. **Native API versioning:** enable a strategy in `WebMvcConfigurer.configureApiVersioning` (header `X-API-Version` or media-type param) and ship `GET /books/{id}` as `version = "1.0"` and `"1.1"` (v1.1 adds a field). Mention RFC 9745 deprecation support.
2. **HTTP interface client:** the price-check adapter per ADR-008 — `@GetExchange` interface + `@ImportHttpServices`.
3. **In-framework resilience:** `@Retryable` (backoff + jitter) on the price-check adapter; `@ConcurrencyLimit` where a bulkhead makes sense.
4. **Virtual threads:** `spring.threads.virtual.enabled=true`; then, for contrast, convert the price-check orchestration to a `suspend` flow with `coroutineScope` fan-out and articulate ADR-007 out loud.
5. **RestTestClient** (new in 7.0) in one adapter test.

### 🌶️ Stretch (30–60 min each)
- **Actuator hardening:** expose only `health,info,metrics`; custom `HealthIndicator` (book count > 0).
- **OpenAPI:** `springdoc-openapi`; annotate one endpoint fully.
- **Caching:** `@Cacheable`/`@CacheEvict` on book reads; explain why caching mutable stock is dangerous.
- **Spring Modulith:** verify module boundaries as an ArchUnit alternative — pairs beautifully with hexagonal packages.
- **Kotlin Serialization module:** Boot 4 ships `spring-boot-kotlin-serialization-starter`; swap Jackson on one endpoint and compare.

---

# PART IV — INTERVIEW ARSENAL

## Annotation cheat sheet (self-quiz: cover the right column)

| Annotation | One-liner |
|---|---|
| `@SpringBootApplication` | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@Component/@Service/@Repository` | Stereotypes; `@Repository` also translates persistence exceptions |
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@Valid` vs `@Validated` | Jakarta trigger vs Spring's (groups, class-level method validation) |
| `@Transactional` | Proxy-based tx boundary; self-invocation & rollback rules |
| `@Version` | Optimistic locking column |
| `@ConfigurationProperties` | Typed, constructor-bound config binding |
| `@RestControllerAdvice` | Global exception → ProblemDetail mapping |
| `@field:NotBlank` | Kotlin use-site target → annotation lands on the backing field |
| `@ServiceConnection` | Wires a Testcontainer into the context automatically |
| `@HttpExchange` / `@GetExchange` | Declarative HTTP interface client (Spring-native "Retrofit") |
| `@ImportHttpServices` | Boot 4: one-annotation registration of HTTP interfaces |
| `@Retryable` / `@ConcurrencyLimit` | Framework 7 in-core resilience (retry w/ backoff; bulkhead) |
| `@GetMapping(version = "1.1")` | Framework 7 first-class API versioning |

## Rapid-fire questions (answer out loud while building)

*Original fourteen:*
1. Why does Kotlin need `plugin.spring` and `plugin.jpa`? What breaks without each?
2. Why not data classes for JPA entities? Where *should* you use them?
3. Constructor vs field injection — three reasons, plus the Kotlin-specific one.
4. What does `@field:` do and what silently fails without it?
5. How does `@Transactional` work? The self-invocation pitfall?
6. Optimistic vs pessimistic locking — when each; `@Version` conflict behavior?
7. Why Flyway + `ddl-auto=validate` over `ddl-auto=update`?
8. `Page` vs `Slice`? What extra SQL does `Page` run? (count query)
9. How do you trace one request across log lines? (MDC / correlation id)
10. N+1: how to detect; two fixes (fetch join, `@EntityGraph`).
11. Platform types; what do `-Xjsr305=strict` / JSpecify change in Boot 4?
12. MockK vs Mockito for Kotlin — why?
13. `RestClient` vs `RestTemplate` vs `WebClient` — which for a new blocking MVC app?
14. Coroutines vs virtual threads for a Spring MVC service.

*New — architecture round:*
15. State the hexagonal dependency rule in one sentence. How do you *enforce* it?
16. Input ports vs output ports — which are you willing to drop under pressure, and why?
17. Why is `FooServiceImpl` with one implementation an anti-pattern in Spring/Kotlin?
18. Your domain model and JPA entities are separate classes. Defend the mapping cost.
19. Where does `@Transactional` live in a hexagonal app and what's the purist objection?
20. How would you add a GraphQL API to this system? What changes? (Answer: one new inbound adapter; nothing else.)
21. Retrofit vs Spring HTTP interfaces on the server — what did Framework 6/7 change?
22. What's new in Spring Framework 7 / Boot 4? (API versioning, JSpecify null-safety, in-framework resilience, Jackson 3, modularized starters, `RestTestClient`.)
23. Spring's `Pageable` at the use-case boundary: leak or pragmatism? Argue both sides.
24. How do virtual threads change the argument for `@ConcurrencyLimit` / bulkheads?

# PART V — DEFINITION OF DONE

- [ ] App boots via Docker Compose Postgres; Flyway migrates; `ddl-auto=validate`
- [ ] `domain` package has **zero** Spring/JPA imports — proven by an ArchUnit test
- [ ] Domain + application layers fully unit-tested without a Spring context
- [ ] Full Book CRUD through the web adapter: DTOs, validation, 201/204/404/409
- [ ] Every error is an RFC 9457 `ProblemDetail` from one advice class
- [ ] `requestId` in every log line via MDC
- [ ] Paged + filtered listing, size capped, port speaks `PageQuery` not `Pageable`
- [ ] Transactional order placement; two-thread test proves optimistic-lock 409
- [ ] Price check goes through an HTTP interface client behind `PriceCheckPort`, with `@Retryable`
- [ ] One endpoint served in two versions via Spring 7 native API versioning
- [ ] Virtual threads enabled; you can argue ADR-007 for two minutes unprompted
- [ ] ≥1 MockK unit test, ≥1 `@WebMvcTest`, ≥1 Testcontainers integration test, ≥1 ArchUnit test
- [ ] You can answer all 24 rapid-fire questions without looking

---

# References (research trail)

**Spring Framework 7 / Boot 4 features**
- InfoQ — *Spring Framework 7 and Spring Boot 4 Deliver API Versioning, Resilience, and Null-Safe Annotations*: https://www.infoq.com/news/2025/11/spring-7-spring-boot-4/
- InfoQ — *The Spring Team on Spring Framework 7 and Spring Boot 4* (interview): https://www.infoq.com/articles/spring-team-spring-7-boot-4/
- Spring Framework 7.0 Release Notes: https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes
- Dan Vega — *What's New in Spring Framework 7 and Spring Boot 4*: https://www.danvega.dev/blog/spring-boot-4-is-here
- Baeldung — *Spring Boot 4 & Spring Framework 7 – What's New*: https://www.baeldung.com/spring-boot-4-spring-framework-7

**Hexagonal architecture with Spring + Kotlin**
- Arho Huttunen — *Hexagonal Architecture With Spring Boot* (incl. the JPA tension & pagination discussion): https://www.arhohuttunen.com/hexagonal-architecture-spring-boot/
- Hieu Nguyen (Medium) — *Hexagonal Architecture in Practice: Spring Boot + Kotlin*: https://medium.com/@hieunv/understanding-hexagonal-architecture-through-a-practical-application-2f2d28f604d9
- Gaurav Kumar (Medium) — *Hexagonal Architecture in Kotlin Spring: A Quick Guide*: https://medium.com/@slashgkr/hexagonal-architecture-in-kotlin-spring-a-quick-guide-a5ecdc3c4c95
- Reference repo (multi-module, ArchUnit-enforced): https://github.com/dustinsand/hex-arch-kotlin-spring-boot
- codecentric — *Spring Modulith with Kotlin and hexagonal architecture*: https://www.codecentric.de/en/knowledge-hub/blog/modularization-the-easy-way-spring-modulith-with-kotlin-and-hexagonal-architecture

**Concurrency: coroutines vs virtual threads**
- JDriven — *From Java to Kotlin Part X: Virtual Threads and Coroutines* (2026): https://jdriven.com/blog/2026/02/Java-To-Kotlin-Series-10-Threads
- Jorge Gonzalez (Medium) — *Comparing Kotlin Coroutines and Java Virtual Threads with Spring Boot Examples*: https://medium.com/@jorgegfx/harnessing-modern-concurrency-comparing-kotlin-coroutines-and-java-virtual-threads-with-spring-cf79e234b892
- Kotlin Discussions — *Spring, Coroutines, Virtual Threads*: https://discuss.kotlinlang.org/t/spring-coroutines-virtual-threads/29949
- Benchmarks repo (coroutine vs Reactor vs virtual threads): https://github.com/gaplo917/coroutine-reactor-virtualthread-microbenchmark

**HTTP clients**
- MEsfandiari (Medium) — *The End of RestTemplate: Spring RestClient vs. WebClient vs. Retrofit in 2026*: https://medium.com/@mesfandiari77/the-end-of-resttemplate-spring-restclient-vs-webclient-vs-retrofit-in-2026-0ca96c463544

Build it once without AI autocomplete, narrating your decisions out loud like it's the interview. 🚀
