# 📚 Bookstore API — Software Design Document
### Kotlin 2.3 · Java 21 · Spring Framework 7 · Spring Boot 4.1 · Hexagonal Architecture

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

This codebase is *also* a **Kotlin-first learning vehicle** targeting Spring Boot 4 / Spring Framework 7 APIs. Plain Kotlin modeling, structured concurrency, `Deferred<T>`, cancellation, dispatchers, Flow, and coroutine testing are learned before Spring enters the curriculum. The domain then forces contact with the features that dominate real API work:

| Real-world concern | Feature it forces you to exercise |
|---|---|
| Two aggregates with ownership | Spring Data JDBC aggregate mapping, `@MappedCollection`, explicit cross-aggregate references |
| Money + identity invariants | `BigDecimal`, validation, unique constraints |
| "Last copy" races | `@Transactional`, `@Version` optimistic locking |
| Multiple clients, long-lived contract | DTOs, API versioning (native in Spring 7), ProblemDetail |
| An external dependency | HTTP interface clients, `@Retryable`, resilience |
| Someone has to run it | Actuator, MDC/correlation IDs, typed configuration |
| Someone has to change it safely | Hexagonal boundaries, MockK, Testcontainers, ArchUnit |
| Kotlin concurrency before frameworks | `Job`, `Deferred<T>`, `join`, `await`, `awaitAll`, structured cancellation, supervision, `runTest` (K0–K2) |
| Blocking vs suspending vs non-blocking | suspend MVC + adapter-owned `withContext` vs virtual threads vs Spring Data R2DBC (M11) |

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
  - verifies prices against the external service before opening the database transaction,
  - checks and **atomically decrements stock** for every line item,
  - computes the total server-side (client-supplied totals are never trusted),
  - persists order + items in **one transaction** — any failure rolls back everything,
  - resolves concurrent conflicts on the same book via optimistic locking → `409 Conflict`.
- `GET /api/v1/orders/{id}` returns the order with its items (no lazy-loading leaks).
- Order lifecycle: `CREATED → PAID | CANCELLED` (stored through an explicit stable-string converter, never an ordinal).

### 3.3 Operational surface
- Health, info and metrics via Actuator (only those three exposed).
- Every log line carries a request-scoped correlation id (`requestId` in MDC).
- Machine-readable errors: every non-2xx response is an RFC 9457 `ProblemDetail`.
- One endpoint demonstrates Spring 7 **native API versioning** (`version = "1.1"` mapping) so the evolution story is proven, not theoretical.

### 3.4 Coroutine application and runtime comparison (K0–K2, M11)
- K0–K2 establish the pure Kotlin model before Spring: `Deferred<T>.join()` vs `await()`, `awaitAll`, structured failure, supervision, cancellation, dispatcher ownership, cold Flow, and `runTest`.
- `GET /api/v1/books/{isbn}/enriched` — book + verified price + rating, fanned out with `suspend` + `coroutineScope`/`async`; the blocking `CompletableFuture`/virtual-thread version is the comparison.
- `GET /api/v1/books/stream?author=` — streaming catalog search returning `Flow<BookResponse>` as NDJSON.
- A second simulated external, the **rating service**, exists so the fan-out has real siblings to race and cancel.
- An `r2dbc-comparison` branch replicates the catalog read path on **Spring Data R2DBC** (`CoroutineCrudRepository`, suspend + `Flow` end to end) — see ADR-012.

### 3.5 Core-service non-goals
- Real payments, shipping, tax, multi-currency — `PAID` is a status flip in the core application, not a gateway integration.
- Authentication/authorization (Spring Security is its own refresher project; the architecture leaves a clean seam for it in the web adapter).
- Kafka, distributed Sagas, and event sourcing are not runtime dependencies of the core monolith.
- Frontend, admin UI, reporting.

Cutting these is what keeps the core service honest: **small surface, production-grade depth.**

### 3.6 Companion learning systems (separate scopes)

The split curriculum deliberately grows beyond the core SDD without pretending every technology belongs in one deployable application:

| Companion | Business purpose | Main techniques |
|---|---|---|
| Fulfillment API | Own shipment/tracking lifecycle and live customer updates | WebFlux, Kotlin `Flow`, SSE, R2DBC, carrier webhooks |
| Kafka/EDA track | Move committed facts from modules to independent consumers | Spring Modulith, publication registry/outbox, Kafka, inbox/idempotency, consumer groups |
| Distributed checkout | Coordinate order, inventory, simulated payment authorization, and fulfillment | Choreographed and orchestrated Sagas, compensation, deadlines, recovery |
| Fulfillment event-sourcing lab | Reconstruct a naturally chronological aggregate and rebuild query models | Append-only event store, expected version, CQRS projections, snapshots/evolution |

The simulated payment service accepts only a provider token; card data is never in scope. These companion systems have their own problem statements, acceptance criteria, and definitions of done under [`docs/bookstore-tour`](bookstore-tour/README.md).

## 4. Actors and external systems

| Actor | Interaction |
|---|---|
| Catalog manager (via admin client) | CRUD on books |
| Storefront / ordering client | Browses catalog, places orders |
| Operations / SRE | Actuator endpoints, structured logs |
| **External price-check service** (simulated) | Outbound HTTP call during order placement — exists to force an outbound-adapter + resilience story |
| **External rating service** (simulated, M11) | Second outbound call — exists so parallel fan-out (`async` vs `CompletableFuture`) has real siblings to race and cancel |
| PostgreSQL 18 | Owned exclusively by this service; schema evolved only via Flyway |

## 5. Quality attributes (the "-ilities" that drove the architecture)

1. **Consistency under concurrency** — the defining requirement. Optimistic locking + single-transaction order placement.
2. **Evolvability** — business rules isolated from frameworks; new delivery mechanisms or storage engines are new adapters, not rewrites. Contract versioned.
3. **Testability** — domain logic testable with plain JUnit (no Spring context, milliseconds); each adapter testable in isolation; the whole thing verifiable against a *real* Postgres via Testcontainers.
4. **Observability** — correlation ids, standard error contract, health/metrics.
5. **Comprehensibility** — a new developer (or you, in six months) can find any rule by asking "is this business or plumbing?" and opening exactly one package.

---

# PART II — ARCHITECTURE

## 6. Architectural style: Hexagonal (Ports & Adapters), package-enforced

The system follows **hexagonal architecture** (Alistair Cockburn, 2005), a practical Clean Architecture shape for a Spring service. The **domain** is plain Kotlin. Application use cases own input/output ports and intentionally use Spring's `@Service` stereotype; adapters provide HTTP, JDBC, and remote-client implementations. Dependencies still point inward: allowing a focused Spring annotation is not permission for application code to import adapter types.

```
            ┌──────────────────────────────────────────────────┐
  driving   │                    adapters                      │  driven
  (input)   │  ┌────────────────────────────────────────────┐  │  (output)
            │  │              application                   │  │
 REST ──────┼─▶│  input ports   ┌──────────┐  output ports  │◀─┼────── JDBC/Postgres
 controller │  │  (use cases)   │  domain  │  (interfaces)  │  │
            │  │                │ Book,    │                │◀─┼────── WebClient →
 (future:   │  │  PlaceOrder    │ Order,   │ BookRepository │  │       price service
  GraphQL,  │  │  ManageBooks   │ rules    │ PriceGateway   │  │
  CLI, MQ)  │  └────────────────└──────────┘────────────────┘  │
            └──────────────────────────────────────────────────┘
                        dependency direction: always inward →◀←
```

**The dependency rule, concretely:**

| Layer | May depend on | Must NOT depend on |
|---|---|---|
| `domain` | Kotlin stdlib, `java.time`, `java.math` | Spring, Jakarta, Spring Data, Jackson, application/adapters |
| `application` | `domain`, port contracts, `kotlinx-coroutines-core`, Spring stereotypes such as `@Service` | web, persistence rows/repositories, HTTP client types, bootstrap |
| `adapter.input.http` | application input ports, DTO/mapping concerns, Spring MVC/validation | output-adapter implementation types |
| `adapter.output.*` | application output ports + domain, Spring Data/client technology | input adapters |
| `bootstrap` | every ring for wiring/configuration only | business rules |

### 6.1 Package layout (single Gradle module — see ADR-002)

```
com.example.kotlinrefresher
├── bootstrap/                              # @SpringBootApplication, @Configuration, @Bean, properties
├── catalog/
│   ├── domain/                             # Book, Isbn, stock behavior/errors; zero Spring
│   ├── application/
│   │   ├── port/input/                     # ManageBooksUseCase, QueryBooksUseCase
│   │   ├── port/output/                    # BookRepositoryPort
│   │   ├── usecase/                        # @Service implementations
│   │   └── error/                          # BookNotFound, DuplicateIsbn
│   └── adapter/
│       ├── input/http/                     # @RestController, DTOs, advice
│       └── output/persistence/jdbc/         # @Repository adapter, BookRow, internal Spring Data repo
├── ordering/
│   ├── domain/                             # Order, OrderItem, order invariants
│   ├── application/
│   │   ├── port/input/                     # PlaceOrderUseCase
│   │   ├── port/output/                    # PriceCheckPort, OrderCommitPort, RatingPort
│   │   └── usecase/                        # @Service PlaceOrderService, EnrichBookService
│   └── adapter/
│       ├── input/http/                     # @RestController OrderController
│       └── output/
│           ├── persistence/jdbc/            # @Repository + imperative @Transactional delegate
│           ├── pricing/                     # @Component HTTP client adapter
│           └── rating/                      # @Component rating adapter
└── shared/                                  # only genuinely shared domain/application primitives
```

This feature-first layout keeps a growing codebase navigable without losing hexagonal rings. `input`/`output` are used instead of `in`/`out` because `in` is a hard Kotlin keyword. Boundaries are **enforced by ArchUnit tests** (M9), not by convention. On the `r2dbc-comparison` branch, the persistence output adapter changes while input ports and domain behavior remain stable.

### 6.2 Spring annotation policy

| Annotation | Allowed role |
|---|---|
| `@RestController` / `@RestControllerAdvice` | HTTP input adapters |
| `@Service` | Application use-case implementations |
| `@Repository` | Persistence output adapters; semantic role/scanning here, with synchronous Spring Data proxies providing JDBC exception translation |
| `@Component` | Non-persistence output adapters and infrastructure collaborators when no specific stereotype fits |
| `@Configuration`, `@Bean`, `@ConfigurationProperties` | Bootstrap/configuration |
| Spring/Jakarta/Data annotations | **Never in domain** |

Spring Data repository interfaces are discovered automatically and stay `internal`; adding `@Repository` to them is redundant. Constructor injection is mandatory. No field injection, `lateinit`, service locator, or static application context access.

---

## 7. Architecture Decision Records

Each ADR: context → decision → why → what you learn defending it.

### ADR-001 · Hexagonal over classic three-layer — with eyes open

**Context.** The default Spring tutorial layout (`controller → service → repository`) works, but a global service layer easily accumulates HTTP, persistence, and business concerns, and the domain ends up shaped like database rows.

**Decision.** Feature-first ports & adapters, with a **framework-free domain** and Spring-annotated application/adapters according to section 6.2. Domain rules remain testable with plain JUnit in milliseconds; `@Service` use cases are also instantiated directly in unit tests without a Spring context.

**Honest cost accounting:** hexagonal buys replaceable technology, isolated tests and deferred decisions — and charges mapping boilerplate, more files, and team discipline. For trivial CRUD it is overkill; this project has concurrency rules, remote dependencies, and two persistence variants, so the ports perform real work.

**🎤 Talking point:** "Hexagonal isn't about the folder names. It's one rule — *dependencies point inward* — plus interfaces at the boundaries so I can compile and test the business core without booting a framework."

### ADR-002 · Single Gradle module + ArchUnit, not multi-module

**Context.** Strict implementations split `core` / `api` / `data` into separate Gradle modules so the compiler physically prevents illegal imports. Multi-module is the stronger guarantee but adds build ceremony that distracts from the learning goals.

**Decision.** One module; boundaries enforced by **ArchUnit** rules (`noClasses().that().resideInAPackage("..domain..").should().dependOnClassesThat().resideInAPackage("org.springframework..")` and layer checks). If this grew into a team codebase, promoting packages to modules is a mechanical refactor precisely *because* the dependencies already point the right way.

### ADR-003 · Interfaces between layers: only at architectural boundaries

**Context.** You asked whether to put interfaces between layers. There are two very different practices hiding under that phrase:

1. **Ports** — interfaces that mark an *architectural boundary* (core ↔ outside world).
2. **The `FooService` / `FooServiceImpl` reflex** — an interface for every class, same package, single implementation.

**Decision.** (1) always; (2) never.

- **Output ports are mandatory.** `BookRepositoryPort`, `OrderCommitPort`, and `PriceCheckPort` are Kotlin interfaces *owned by the application layer*; JDBC/R2DBC and HTTP adapters implement them. This is the **Dependency Inversion Principle** doing real work.
- **Input ports (use-case interfaces) are included here for the learning value** — they make the "what does this system do?" list greppable (`PlaceOrderUseCase`) and keep controllers ignorant of implementation classes. Know the counter-argument: many production teams skip them and let controllers call application services directly, because controller → service already points inward. Skipping input ports is a defensible pragmatic trim; skipping output ports collapses the architecture.
- **`ServiceImpl` pairs add nothing**: Spring has proxied concrete classes via CGLIB for a decade (that's what `plugin.spring`/all-open is for), MockK mocks final classes, and a single-implementation interface in the same package is speculative generality. Add an interface *within* a layer only when a second implementation or a boundary actually exists.

**🎤 Talking point:** "DIP is about the *direction of the dependency arrow*, not about the interface keyword. I use interfaces where the arrow needs to flip — at ports — and concrete classes everywhere else."

### ADR-004 · SOLID, mapped to actual code decisions

| Principle | Where it materializes in this codebase |
|---|---|
| **S**RP | Controllers translate HTTP only; use-case services orchestrate one business operation each; adapters translate technology. A change in JSON shape, business rule, or SQL touches exactly one place. |
| **O**CP | Adding a GraphQL API or swapping Postgres for Mongo = new adapter implementing an existing port; zero edits to `domain`/`application`. |
| **L**SP | Every `BookRepositoryPort` implementation (JDBC adapter, R2DBC adapter, test fake) honors the same nullable/suspend contract. Test fakes that cheat it poison tests. |
| **I**SP | Ports are role-shaped and small (`PriceCheckPort` has one method), so no adapter is forced to stub methods it doesn't serve. |
| **D**IP | High-level policy (`PlaceOrderService`) depends on output ports it owns; low-level JDBC/R2DBC/WebClient details implement those abstractions. Constructor injection of `val` references makes the graph explicit. |

### ADR-005 · Domain model separate from Spring Data rows

**Context.** Spring Data JDBC can map Kotlin classes directly, but using the same type for business behavior and storage still lets table concerns, nullable generated ids, versions, and aggregate serialization shape the domain.

**Decision.** Separate them. `catalog.domain.Book` enforces invariants; `BookRow` is an immutable `@Table` data class in the JDBC adapter. Internal extension functions map between them. The R2DBC comparison gets its own row type without changing the domain.

**Consequence accepted:** mapping boilerplate for two aggregates. It is small, explicit, and prevents persistence annotations from crossing into domain.

### ADR-006 · Spring stereotypes are intentional; JDBC transactions remain imperative internally

**Context.** The user wants normal Spring annotation-driven code, while Kotlin coroutines can resume on different threads and JDBC transactions are thread-bound. Remote price calls must not occur while a DB transaction holds connections/locks.

**Decision.** `PlaceOrderService` is an `@Service` with a suspending method. It performs remote quote fan-out, then invokes `OrderCommitPort`. `JdbcOrderCommitAdapter` is an `@Repository`; inside `withContext(jdbcDispatcher)` it calls a separate proxied bean with an imperative `@Transactional` method. That method loads/version-checks aggregates, runs domain behavior, and writes atomically without a suspension point. The R2DBC adapter uses a reactive transaction manager for its suspend implementation.

**🎤 Talking points (unchanged interview gold):**
- Proxy-based ⇒ **self-invocation bypasses the transaction** (`this.otherTxMethod()` never hits the proxy).
- Default rollback: unchecked only. Kotlin has no checked exceptions, but Spring still applies the Java rule *by exception type* — a Kotlin function throwing `IOException` will NOT roll back by default. Know `rollbackFor`.
- `readOnly = true` expresses intent and may enable driver/database optimizations; Data JDBC has no Hibernate dirty-checking session.

### ADR-007 · Concurrency: Kotlin coroutines are the main line; virtual threads are the comparison

**Context (the question you actually asked).** Kotlin coroutines vs Java 21+ virtual threads is *the* 2026 concurrency interview topic — and "pick a winner" is the wrong frame. There are three independent dials: **code shape** (`suspend`/`Flow` signatures vs plain blocking ones), **runtime** (virtual threads make blocking cheap), and **driver** (JDBC blocks; R2DBC doesn't). A learning project should turn each dial separately and diff the results.

**Decision.** K0–K2 teach coroutines before Spring. Suspend MVC/use-case/output-port contracts are the main line. M11 applies them to Spring and then builds the blocking virtual-thread comparison.

- **Track A — Kotlin-first main line:** suspend MVC and ports, `coroutineScope { async {} }` fan-out, adapter-owned `withContext(jdbcDispatcher)` for Data JDBC, Flow only for streaming contracts, and R2DBC where IO becomes truly non-blocking.
- **Track B — imperative comparison:** blocking MVC/Data JDBC with `spring.threads.virtual.enabled=true`, `CompletableFuture` only for the explicit comparison, and ordinary pagination.

**The fan-out pair, condensed** (M11 builds it in full):

```kotlin
// Track A — structured: rating fails ⇒ price is cancelled, scope rethrows. Guaranteed.
suspend fun enrich(isbn: String): EnrichedBook = coroutineScope {
    val price  = async { priceCheckPort.currentPrice(Isbn(isbn)) }
    val rating = async { ratingPort.rating(isbn) }
    EnrichedBook(price.await(), rating.await())
}
```

```kotlin
// Track B — unstructured: rating fails ⇒ price keeps running unless you cancel it yourself.
fun enrich(isbn: String): EnrichedBook {
    val price  = CompletableFuture.supplyAsync({ blockingPriceCheckPort.currentPrice(Isbn(isbn)) }, vtExecutor)
    val rating = CompletableFuture.supplyAsync({ blockingRatingPort.rating(isbn) }, vtExecutor)
    return EnrichedBook(price.join(), rating.join())
}
```

Track B uses explicit imperative comparison ports; a `CompletableFuture` supplier cannot call a suspending port. The executor is injected from bootstrap and closed by the container, not allocated inside the service.

**How JDBC fits Track A:** coroutines do not make JDBC non-blocking. The `@Repository` adapter owns an injected `Dispatchers.IO.limitedParallelism(n)` view, while its imperative `@Transactional` delegate stays on one thread. `Dispatchers.IO` defaults to 64 or the processor count, whichever is larger, and its limited views are elastic; the connection pool remains the hard resource boundary. Track B instead uses virtual threads to reduce the cost of the same blocking waits.

**What Track A buys, and its named costs:**
- **Cancellation by construction** (the code pair above). Java's answer, `StructuredTaskScope`, is still in preview as of Java 25.
- **Streaming:** `Flow<T>` is cold, backpressured, cancellable; on NDJSON/SSE Spring serializes elements as they arrive — on plain `application/json` the flow is silently *collected* first. The media type decides.
- **Composition with virtual threads:** `Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()` — suspend-shaped code, Loom-cheap blocking. The tracks stack; they don't compete.
- **Costs:** `suspend` colors contracts and `kotlinx-coroutines-core` enters application. MVC needs `kotlinx-coroutines-reactor`. Boot/Micrometer automatic context propagation is enabled for observability; custom MDC keys need an accessor or explicit `MDCContext`. JDBC transactions remain imperative behind the adapter; R2DBC uses reactive transaction context.

**🎤 Talking point:** "Suspending is a property of code, virtual threads of the runtime, and blocking of the driver. My main line uses structured coroutines, isolates JDBC blocking in an adapter, and changes only that adapter when R2DBC removes the block."

### ADR-008 · HTTP client: WebClient-backed reactive interface behind a suspend port; RestClient comparison

**Context (your other question).** For the outbound price-check call the candidates are Retrofit, `RestTemplate`, `WebClient`, `RestClient`, and Spring 6/7's declarative HTTP interfaces.

**Decision.** The application owns a suspending `PriceCheckPort`. The WebClient-backed technical HTTP interface (`@GetExchange`) returns `Mono<PriceQuote>` and is registered as a Boot HTTP-service group. The adapter awaits that publisher, so Reactor stays outside the application boundary. This shape also lets Spring Framework's reactive `@Retryable` interceptor decorate the actual asynchronous pipeline. M11 builds the blocking RestClient-backed interface for comparison.

**The elimination round:**
- **`RestTemplate`** — maintenance mode; feature-frozen. Legacy code only.
- **`WebClient`** — correct here because the port suspends; Spring MVC may use a reactive client without exposing Reactor types from controllers.
- **`RestClient`** — correct for an intentionally blocking port/virtual-thread track.
- **Retrofit** — good on Android, but Spring HTTP interfaces provide native server-side integration, service groups, testing, and version insertion.
- Spring 7's **`@Retryable`** lives on a separate proxied adapter collaborator returning `Mono` and requires `@EnableResilientMethods`.
- A coroutine `Semaphore.withPermit` is the suspend main line's bulkhead. `@ConcurrencyLimit` belongs on the synchronous RestClient/virtual-thread comparison; putting it on `suspend fun` would release the invocation guard at suspension rather than hold a permit for the operation.

```kotlin
interface PriceCheckClient {
    @GetExchange("/prices/{isbn}")
    fun currentPrice(@PathVariable isbn: String): Mono<PriceQuote>
}

@Configuration
@ImportHttpServices(group = "pricing", types = [PriceCheckClient::class])
class HttpClientsConfig
```

The application core never sees this interface — it sees `PriceCheckPort`; `PriceCheckAdapter` uses `awaitSingle()` inside a coroutine `Semaphore.withPermit` block. A separate `@Component` owns the reactive `@Retryable` method so the call crosses a proxy. Swap Retrofit in later and *nothing* inside the hexagon changes — which is the architecture proving its worth.

Add `spring-boot-starter-webclient` and configure `spring.http.serviceclient.pricing.base-url` plus timeouts in bootstrap. The application sees only `PriceCheckPort`; it never imports `Mono`, the HTTP interface, or WebClient.

### ADR-009 · API contract: DTOs, ProblemDetail, capped paging, native versioning

- **DTO boundary is absolute.** Entities/domain objects never serialize to JSON. Request/response types are Kotlin `data class`es (here they're *ideal* — value semantics, `copy()` for test builders), mapped via extension functions.
- **Errors** are RFC 9457 `ProblemDetail` from one `@RestControllerAdvice` — a standard, machine-readable contract (`type`, `title`, `status`, `detail` + a field-error map for validation failures). Mention that 9457 supersedes 7807 and you sound current.
- **Paging** accepts `Pageable`, returns mapped `Page<BookResponse>`, caps size at 100 (pagination-DoS). Hexagonal nuance worth knowing: passing Spring's `Pageable` through the input port leaks framework into the core — the strict fix is your own small `PageQuery` type; teams often accept the leak for query-only paths. We define `PageQuery` (it's ~5 lines) and say why.
- **Versioning**: URL prefix `/api/v1` as the coarse contract, *plus* one endpoint demonstrating Spring 7's first-class versioning — `@GetMapping(version = "1.1")` with an `ApiVersionConfigurer` strategy (path/header/query/media-type all supported, with RFC 9745 deprecation signaling). This feature is new headline material in Framework 7; showing it in a portfolio project is cheap and differentiating.

### ADR-010 · Spring Data JDBC, Flyway, and Kotlin persistence discipline

- **Flyway owns the schema.** Data JDBC does no DDL generation/validation; Testcontainers integration tests prove mappings against migrated Postgres.
- Money = `NUMERIC(10,2)` ↔ `BigDecimal`. Never `Double`.
- JDBC rows are immutable `@Table` data classes inside the adapter; domain objects remain separate. `@MappedCollection` expresses aggregate ownership; references across aggregates are ids.
- Spring Data `@Version` ⇒ optimistic locking ⇒ concurrent last-copy purchase resolves as `OptimisticLockingFailureException` → `409`.
- The main line needs `plugin.spring` for proxied annotated beans, not `plugin.jpa`. The JPA plugin appears only in the optional ORM comparison.
- **Null-safety at the boundary:** Boot 4 / Framework 7 completed the migration to **JSpecify** annotations, and Kotlin 2.1+ translates them into real Kotlin nullability — Spring API returns are `User?`, not platform types. Keep `-Xjsr305=strict` for remaining third-party libs. This is *the* Kotlin-specific "why Boot 4 matters" line.

### ADR-011 · Testing strategy = the architecture, verified

| Ring | Tooling | What it proves |
|---|---|---|
| Kotlin runway | `kotlinx-coroutines-test` + JUnit Jupiter | join/await, cancellation, supervision, Flow, virtual time |
| Domain | Plain JUnit Jupiter — **no Spring, no mocks** | Business invariants; runs in ms |
| Application | MockK fakes/mocks of *ports* | Suspend orchestration with `coEvery`/`coVerify`; business errors |
| Web adapter | `@WebMvcTest` + mocked input port | Status codes, JSON shape, validation 400s, ProblemDetail shape |
| Persistence adapter | `@DataJdbcTest` + **Testcontainers Postgres** via `@ServiceConnection` | Real SQL, aggregate mapping, constraints, versions |
| Whole hexagon | `@SpringBootTest` + Testcontainers | Place-order flow incl. rollback & optimistic-lock 409 |
| Architecture | **ArchUnit** | The dependency rule itself |

MockK over Mockito stays the Kotlin answer: final classes, `suspend` functions, extension functions, Kotlin DSL.

### ADR-012 · The R2DBC branch line: same domain, non-blocking driver (M11)

**Context.** Coroutines over Data JDBC still isolate rather than eliminate blocking. The `r2dbc-comparison` branch makes the catalog read path non-blocking with Spring Data R2DBC.

**Decision.** Data JDBC remains the main line. The branch swaps the persistence adapter for `CoroutineCrudRepository`, suspend derived queries, and Flow returns. Flyway keeps a JDBC URL because startup migrations need not be reactive. Do not configure JDBC and R2DBC repositories for the same aggregate in the learning branch.

```kotlin
@Table("books")                       // Spring Data annotations — jakarta.persistence appears nowhere
data class BookR2dbcRow(
    @Id val id: Long? = null,
    val isbn: String, val title: String, val author: String,
    val price: BigDecimal, val stock: Int,
    @Version val version: Long = 0,
)

interface CoroutineBookRepository : CoroutineCrudRepository<BookR2dbcRow, Long> {
    suspend fun findByIsbn(isbn: String): BookR2dbcRow?
    fun findByAuthorContainingIgnoreCase(author: String): Flow<BookR2dbcRow>
}
```

**What flips relative to ADR-010:**
- The row stays immutable; the repository signature and driver change.
- Explicit aggregates, joins, and saves remain explicit — R2DBC changes IO, not relational modeling.
- **`@Version` optimistic locking survives** — the last-copy race still resolves as one success + one `OptimisticLockingFailureException` → the same 409 contract. The consistency story is driver-independent, which is precisely why it lives behind a port.
- **`@Transactional` works on `suspend` functions here**, because the reactive transaction manager propagates through the coroutine context instead of a ThreadLocal — order placement stays atomic on this track too.
- Ports on this track are suspend/`Flow`-shaped — ADR-007's "color reaches the contract" lesson, now enforced by the compiler.

---

# PART III — BUILD PLAN (the learning curve, in order)

The sequence is intentionally Kotlin-first: K0–K2 teach language/concurrency with no Spring context; M1 applies the language to a pure domain; M2 introduces Spring-annotated application services; only then do adapters arrive.

The executable lesson sequence is split into context-first files under [`bookstore-tour`](bookstore-tour/README.md). This SDD remains the core service contract; the WebFlux and advanced tracks are companion contracts rather than extra milestones hidden inside M11.

## Tech stack (current as of mid-2026)

| Piece | Choice | Why |
|---|---|---|
| Spring Boot | **4.1** (Framework 7) | Current project; Java 17+ baseline, Java 21 toolchain |
| Kotlin | 2.3 | Current project; K2 compiler; JSpecify interop |
| Build | Gradle Kotlin DSL | Type-safe, the Kotlin-shop default |
| Persistence | Spring Data JDBC | Kotlin-friendly immutable aggregate rows; blocking isolated in output adapters |
| DB | PostgreSQL 18 (Docker Compose, auto-detected via `spring-boot-docker-compose`) | Matches `.env.example` |
| Migrations | Flyway | Owns all DDL; Data JDBC does not generate schema |
| Testing | JUnit Jupiter + `kotlinx-coroutines-test` + MockK + Testcontainers + ArchUnit | |
| Coroutines (K1 onward) | core, test, reactor; SLF4J bridge only for explicit custom MDC | suspend/Flow in MVC; structured concurrency and virtual-time tests |
| Non-blocking DB (M11 branch) | `spring-boot-starter-data-r2dbc` + `r2dbc-postgresql` | `CoroutineCrudRepository`; Flyway keeps a JDBC url |

Companion applications add WebFlux/R2DBC, Spring Modulith, Spring Kafka, and separate datastore/broker infrastructure only in the lesson where the business problem requires them. MVC/JDBC and WebFlux/R2DBC server stacks are not collapsed into one ambiguous starter application.

Bootstrap at start.spring.io → Kotlin, Gradle-Kotlin → **Web, Data JDBC, Validation, PostgreSQL, Flyway, Actuator, Testcontainers, Docker Compose support**.

```kotlin
plugins {
    id("org.springframework.boot") version "4.1.x"
    id("io.spring.dependency-management") version "1.1.x"
    kotlin("jvm") version "2.3.x"
    kotlin("plugin.spring") version "2.3.x"  // ⭐ all-open for annotated/proxied Spring beans
}
kotlin {
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict", "-Xannotation-default-target=param-property")
    }
}
```

```yaml
# compose.yaml
services:
  postgres:
    image: postgres:18-alpine
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

### ✅ K0 — Pure Kotlin modeling runway
**Refreshes:** nullability, expressions, smart casts, `data`/regular/`value`/sealed classes, immutability, extension functions, collection transformations.

Build `Isbn`, `Money`, a small identity-bearing class, and a sealed decision type. Test them with no Spring and no coroutines.

### ✅ K1 — Structured concurrency runway
**Refreshes:** `suspend`, `CoroutineScope`, `launch`/`Job`, `async`/`Deferred<T>`, `join`, `await`, `awaitAll`, `coroutineScope`.

Tests must prove result retrieval, completion-only waiting, first failure, sibling cancellation, and the absence of `GlobalScope`/unowned tasks.

### ✅ K2 — Context, cancellation, Flow, and coroutine testing
**Refreshes:** injected dispatcher ownership, `withContext`, `Dispatchers.Default` vs blocking IO, `supervisorScope`, timeouts, cancellation, cold Flow/operators, `runTest` and virtual time.

Only after K2 may a Spring controller or adapter use `suspend`/`Flow`.

---

### ✅ M0 — Walking skeleton + typed configuration
**Refreshes:** `@SpringBootApplication`, profiles, `@ConfigurationProperties`, Docker Compose integration.

1. Reconcile the checked-in Compose inputs: define or remove the referenced Redis image and replace PostgreSQL's invalid list-of-maps `environment` form. `docker compose --env-file .env.example config --quiet` must pass before boot is claimed.
2. App boots against Postgres; Flyway runs V1; a `@DataJdbcTest` later proves the mapping against the migrated schema.
3. Bind config to an immutable class and enable with `@ConfigurationPropertiesScan`:
```kotlin
@ConfigurationProperties(prefix = "bookstore.orders")
data class OrderProperties(val maxItemsPerOrder: Int = 10)
```
4. Add `application-local.yml`; run with the profile.

**🎤** Constructor-bound `@ConfigurationProperties` data classes > `@Value`: type-safe, immutable, validatable (`@Validated` + jakarta annotations), metadata-discoverable.

### ✅ M1 — Domain core (pure Kotlin, zero Spring)
**Refreshes:** Kotlin design muscle — `data class` vs class, sealed hierarchies, `init` invariants, `require`, value semantics.

1. Write feature domains: `catalog.domain.Book` owns stock behavior; `ordering.domain.Order`/`OrderItem` own order invariants.
2. Domain behavior failures (for example insufficient stock) stay in domain; not-found and duplicate-use-case failures live in `application.error`.
3. **Plain JUnit tests** — no Spring context, no mocks. `Order` math, stock rules, invariant violations.

**The point:** this package would compile in a Ktor app, a CLI, or a Lambda unchanged. That's the hexagon's core, demonstrated.

### ✅ M2 — Ports + application services
**Refreshes:** interfaces-as-contracts, constructor injection, DIP.

1. Define `port.input` and `port.output` (`BookRepositoryPort`, `OrderCommitPort`, `PriceCheckPort`) with nullable/suspend contracts instead of `Optional` or exposed `Deferred<T>`.
2. Implement use cases as `@Service` classes. Bootstrap maps `OrderProperties` into an application-owned `OrderPolicy`; application never imports the bootstrap type.
3. Unit test the annotated classes directly with MockK `coEvery`/`coVerify` and `runTest` — no Spring context.

**🎤** Constructor injection over field injection: immutability (`val`), can't exist in an invalid state, trivially testable, circular deps fail fast — and in Kotlin, field injection would force `lateinit var`, surrendering null-safety.

### ✅ M3 — Spring Data JDBC output adapter
**Refreshes:** `@Table`, `@Id`, `@Version`, `@MappedCollection`, auditing, immutable row↔domain mapping.

1. `BookRow` as an immutable persistence data class:
```kotlin
@Table("books")
data class BookRow(
    @Id val id: Long? = null,
    val isbn: String, val title: String, val author: String,
    val price: BigDecimal, val stock: Int,
    @Version val version: Long? = null,
)
```
2. Auditing: `@CreatedDate`/`@LastModifiedDate` + `@EnableJdbcAuditing`.
3. The internal `CrudRepository<BookRow, Long>` needs no decorative stereotype. `@Repository BookPersistenceAdapter` implements the output port, owns `withContext(jdbcDispatcher)`, and maps rows/domain.
4. `@DataJdbcTest` + Testcontainers `@ServiceConnection` — real Postgres, no H2 assumptions.

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

Map: `BookNotFoundException`→404 · `DuplicateIsbnException`→409 · `MethodArgumentNotValidException`→400 with field errors · `OptimisticLockingFailureException`→409 · fallback `Exception`→500 generic. Verify the JSON shape.

### ✅ M6 — Logging done properly
**Refreshes:** SLF4J idioms in Kotlin, levels, MDC.

1. Pick and defend a logger idiom: companion-object `LoggerFactory.getLogger(...)` or a reified top-level helper `inline fun <reified T> T.logger(): Logger`.
2. Parameterized messages — `log.info("Book created id={}", id)` — never `"$id"` interpolation (evaluated even when the level is off).
3. `OncePerRequestFilter` that seeds **MDC** with a `requestId`; `%X{requestId}` in the pattern. That's the "trace one request across logs" answer.
4. Enable `org.springframework.jdbc.core` SQL logging only in the local profile; never log credentials or sensitive values.

### ✅ M7 — Pagination, sorting, filtering
**Refreshes:** `Pageable`, `Page<T>`, derived queries, `@Query`.

1. `GET /api/v1/books?page=0&size=20&sort=title,asc` — controller takes `Pageable`, maps `page.map { it.toResponse() }`, cap size at 100.
2. Filters: one derived query (`findByAuthorContainingIgnoreCase`) + one SQL `@Query` — both styles refreshed.
3. Hexagonal wrinkle: the input port speaks a tiny `PageQuery`/`PagedResult` pair, the adapter translates to Spring's `Pageable`. Feel the cost, understand the shortcut (ADR-009).

### ✅ M8 — Orders: transactions + optimistic locking (the meaty one)
**Refreshes:** suspend orchestration, remote fan-out before transactions, proxied imperative `@Transactional`, `@Version`, Data JDBC aggregate ownership.

1. `@Service PlaceOrderService.place()` validates and obtains price/rating data with structured coroutines before opening a DB transaction.
2. `OrderCommitPort` is implemented by an `@Repository` adapter that dispatches blocking work and calls a separate imperative `@Transactional` delegate on the same thread.
3. That delegate loads versioned rows, maps to domain, applies stock behavior, and persists the aggregate atomically.
4. Two-thread test proves exactly one last-copy success and one 409.
5. Stable enum converters and `@MappedCollection` replace JPA ordinals/lazy associations.

### ✅ M9 — Tests completed + ArchUnit
Fill the pyramid per ADR-011 and add the architecture tests:

```kotlin
@AnalyzeClasses(packages = ["com.example.kotlinrefresher"])
class HexagonalRulesTest {
    @ArchTest val domainIsPure = noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "org.springframework..", "jakarta..", "org.springframework.data..", "..adapter..", "..application..")
}
```

### ✅ M10 — Spring 7 / Boot 4 feature tour (the "most-used current API features" pass)
**Refreshes:** what's actually new — the differentiators for 2026 interviews.

1. **Native API versioning:** enable a strategy in `WebMvcConfigurer.configureApiVersioning` (header `X-API-Version` or media-type param) and ship `GET /books/{id}` as `version = "1.0"` and `"1.1"` (v1.1 adds a field). Mention RFC 9745 deprecation support.
2. **HTTP interface client:** the price-check adapter per ADR-008 — `@GetExchange` interface + `@ImportHttpServices`.
3. **Execution-model-aware resilience:** reactive `@Retryable` on the WebClient delegate, coroutine `Semaphore` around the suspend port adapter, `@ConcurrencyLimit` on the synchronous comparison adapter, and `@EnableResilientMethods` in bootstrap.
4. **Virtual threads:** enable them on the explicit imperative comparison profile, not as a hidden replacement for the coroutine main line.
5. **RestTestClient** (new in 7.0) in one adapter test.

### ✅ M11 — Apply the coroutine runway and compare runtimes
**Refreshes:** suspend MVC, structured fan-out, adapter-owned JDBC dispatcher, Flow media-type behavior, virtual-thread comparison, `CoroutineCrudRepository`.

Prereqs: `kotlinx-coroutines-reactor`, Micrometer context propagation, optional `kotlinx-coroutines-slf4j` for custom MDC, and the R2DBC starter/driver only on the comparison branch.

Apply the already-learned Kotlin model first, then diff the imperative alternative:

1. **Suspend end to end** — `GET /api/v1/books/{isbn}/enriched` as `suspend` controller → suspend input port → suspend adapters. The traditional twin lives in the comparison profile with parallel imperative port types; do not call suspend ports through `runBlocking`. *The diff is the signatures*: coroutines color contracts, while switching an already-imperative implementation from platform to virtual threads changes no signatures.
2. **Fan-out** — price + rating in parallel: ADR-007's code pair. *Failure semantics are the headline*: a failed `async` cancels its sibling by construction; the `CompletableFuture` twin leaks it unless you cancel manually.
3. **Bridging Data JDBC** — `@Repository` owns `withContext(jdbcDispatcher)`; the transaction delegate stays imperative and proxied. Compare with a virtual-thread adapter. This isolates blocking; only R2DBC removes it.
4. **Streaming** — the output port returns `Flow<Book>` (note: the method itself is not `suspend` — a cold `Flow` is a recipe, not a result); the controller returns `Flow<BookResponse>` with `produces = APPLICATION_NDJSON_VALUE`. Compare against M7 pagination and argue which *contract* a catalog actually wants. Know the trap: the same handler on `application/json` collects instead of streaming.
5. **R2DBC branch line** (ADR-012) — catalog reads via `CoroutineCrudRepository`: suspend + `Flow` with no `withContext` anywhere, because nothing blocks. Re-run the last-copy race and watch `@Version` still deliver the 409.

Wrap-up drill: prove Boot/Micrometer context propagation after a suspension; add an accessor or explicit `MDCContext` for a custom MDC key.

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
| `@Component` | Generic component-scan stereotype for adapters/infrastructure with no narrower semantic role |
| `@Service` | Application use-case implementation; communicates orchestration intent and supports proxying |
| `@Repository` | Persistence output adapter; communicates role, while Spring Data proxies translate synchronous JDBC failures |
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@Valid` vs `@Validated` | Jakarta validation/cascade trigger vs Spring method validation and validation groups |
| `@Transactional` | Proxy-based tx boundary; self-invocation & rollback rules |
| `@Version` | Optimistic locking column |
| `@ConfigurationProperties` | Typed, constructor-bound config binding |
| `@RestControllerAdvice` | Global exception → ProblemDetail mapping |
| `@field:NotBlank` | Kotlin use-site target → annotation lands on the backing field |
| `@ServiceConnection` | Wires a Testcontainer into the context automatically |
| `@HttpExchange` / `@GetExchange` | Declarative HTTP interface client (Spring-native "Retrofit") |
| `@ImportHttpServices` | Boot 4: one-annotation registration of HTTP interfaces |
| `@Retryable` / `@ConcurrencyLimit` | Framework 7 retry and synchronous invocation bulkhead; execution-model semantics matter |
| `@EnableResilientMethods` | Activates Framework 7 resilient-method interception |
| `@GetMapping(version = "1.1")` | Framework 7 first-class API versioning |

## The two-dialect phrasebook (M11 — same intent, both tracks)

| Intent | Pure Kotlin (coroutines) | Traditional Spring Boot |
|---|---|---|
| Handler that waits on IO | `suspend fun` controller method | plain method on a virtual thread |
| Parallel fan-out | `coroutineScope { async { } }` | `CompletableFuture.supplyAsync(work, vtExecutor)` — someday `StructuredTaskScope` |
| Calling blocking code | adapter-owned `withContext(jdbcDispatcher) { … }` | call it directly; the virtual thread parks |
| Cancel siblings on failure | automatic inside `coroutineScope` | manual `cancel(true)` wiring |
| Many results, streamed | `Flow<T>` + NDJSON/SSE | `Page<T>` batches (or `StreamingResponseBody`) |
| Blocking relational repository | `CrudRepository` (Data JDBC) behind the dispatcher-owning adapter | `CrudRepository` called on a virtual thread |
| Non-blocking relational repository | `CoroutineCrudRepository` (R2DBC) | no JDBC equivalent; changing threads does not remove blocking |
| Bulkhead a remote call | coroutine `Semaphore.withPermit` | `@ConcurrencyLimit(limitString = "…")` around the synchronous invocation |
| Correlation id propagation | Boot/Micrometer propagation; explicit accessor or `MDCContext` for custom MDC keys | `ThreadLocal` remains on the request virtual thread |

## Rapid-fire questions (answer out loud while building)

*Kotlin language and concurrency:*
1. When should a type be a `data class`, regular class, `value class`, or `sealed interface`?
2. What is the difference between `launch`/`Job` and `async`/`Deferred<T>`?
3. What does `Deferred.join()` return, and when must you use `await()` instead?
4. How does `awaitAll()` fail differently from awaiting deferred values one at a time?
5. One `async` fails inside `coroutineScope`: what happens to its sibling? How does `supervisorScope` differ?
6. Where is cancellation cooperative, and why must `CancellationException` be rethrown?
7. Why is `runBlocking` acceptable at a process/test boundary but not in a request handler?
8. Why inject a dispatcher or execution policy instead of hard-coding `Dispatchers.IO` in use cases?
9. Why is a cold `Flow<T>`-returning function normally not itself `suspend`?
10. When should an API return a collection, page, `Flow`, NDJSON, or SSE?

*Spring, annotations, and architecture:*
11. Why does the main line need `plugin.spring` but not `plugin.jpa`?
12. What does `-Xannotation-default-target=param-property` solve, and when is an explicit `@field:` still clearer?
13. Constructor versus field injection: what are the Kotlin-specific advantages?
14. State the hexagonal dependency rule in one sentence. How is it enforced?
15. Why are packages named `input`/`output`, not `in`/`out`, in Kotlin?
16. Why may application use cases be `@Service` while domain types remain annotation-free?
17. When should an output adapter be `@Repository` versus `@Component`?
18. Why is `FooServiceImpl` with one implementation usually ceremony?
19. Where does `@Transactional` live here, and why is the transactional delegate a separate proxied bean?
20. How would a GraphQL adapter reuse the same input ports?

*Persistence, web, and operations:*
21. Why does Flyway own DDL when Spring Data JDBC does no schema generation?
22. How do Spring Data JDBC aggregates and `@MappedCollection` constrain ownership?
23. Why keep domain objects separate from persisted rows despite the mapping cost?
24. Optimistic versus pessimistic locking: when each, and how does `@Version` become a `409`?
25. `Page` versus `Slice`: what extra SQL does `Page` usually require?
26. Why must external price checks finish before the database transaction starts?
27. What belongs in `@RestControllerAdvice`, and what remains a domain/application error?
28. `RestClient` versus `WebClient`: which one supports the suspending main line without hiding a block?
29. What enables Spring Framework resilience annotations, why is `@ConcurrencyLimit` not a suspend-operation bulkhead, and why is retry around the whole order transaction dangerous?
30. How is a correlation id preserved after coroutine suspension, especially for custom MDC keys?

*Runtime comparison:*
31. What does `withContext(jdbcDispatcher)` do to JDBC — and what does it not do?
32. Why must a thread-bound JDBC transaction avoid suspension/thread switches?
33. What decides whether a controller-returned `Flow<T>` streams or is collected?
34. Which dependencies let Spring MVC adapt `suspend` and `Flow` handlers?
35. What changes between Data JDBC and R2DBC, and what stays the same?
36. How do coroutines, virtual threads, and non-blocking drivers change three different dials?

# PART V — DEFINITION OF DONE

- [ ] K0–K2 exercises prove Kotlin modeling, `Job`/`Deferred<T>`, `join`/`await`/`awaitAll`, cancellation, supervision, dispatcher choice, `Flow`, and `runTest`
- [ ] App boots via Docker Compose Postgres 18; Flyway owns and applies every schema change
- [ ] `domain` packages have **zero framework imports** — proven by an ArchUnit test
- [ ] Domain + application layers fully unit-tested without a Spring context
- [ ] Annotation policy is consistent: `@RestController` inbound, `@Service` use cases, `@Repository` persistence adapters, `@Component` other adapters
- [ ] Full Book CRUD through the web adapter: DTOs, validation, 201/204/404/409
- [ ] Every error is an RFC 9457 `ProblemDetail` from one advice class
- [ ] `requestId` in every log line via MDC
- [ ] Paged + filtered listing, size capped, port speaks `PageQuery` not `Pageable`
- [ ] Suspend order orchestration finishes remote calls before a short Data JDBC transaction; a two-request test proves optimistic-lock 409
- [ ] Price check goes through a WebClient HTTP interface behind `PriceCheckPort`; reactive `@Retryable` and coroutine `Semaphore` behavior are tested
- [ ] `@EnableResilientMethods` activates retry/concurrency annotations; `@ConcurrencyLimit` is confined to the synchronous comparison and resource limits are documented
- [ ] One endpoint served in two versions via Spring 7 native API versioning
- [ ] Virtual threads exist only in the imperative comparison profile; you can argue ADR-007 for two minutes unprompted
- [ ] M11: enrichment exists in both dialects — `coroutineScope`/`async` first, `CompletableFuture` on virtual threads second — and you can diff their failure semantics out loud
- [ ] M11: a `Flow<T>` endpoint streams NDJSON, and you can explain the `application/json` collect trap
- [ ] M11: the `r2dbc-comparison` branch serves catalog reads with `CoroutineCrudRepository`, reproduces the `@Version` 409, and keeps Flyway on JDBC
- [ ] ≥1 `runTest`/MockK unit test, ≥1 `@WebMvcTest`, ≥1 `@DataJdbcTest`, ≥1 Testcontainers integration test, and ≥1 ArchUnit test
- [ ] You can answer all 36 rapid-fire questions without looking

**Companion curriculum completion is tracked separately:**

- [ ] Fulfillment API streams durable, authorized tracking updates with WebFlux, R2DBC, `Flow`, SSE, cancellation, and measured subscriber limits
- [ ] Domain events are mapped to versioned integration events and published through a durable publication/outbox mechanism
- [ ] Kafka consumers prove inbox/business idempotency, retry/DLT policy, partition ordering, and independent consumer-group behavior
- [ ] Distributed checkout exposes a `202 Accepted` operation and both Saga variants pass the same happy-path, rejection, timeout, restart, duplicate, and failed-compensation matrix
- [ ] The orchestrated variant persists coordinator state and never models a cross-service Saga as a long-running coroutine
- [ ] Event sourcing remains selective to fulfillment; event-store replay rebuilds aggregate state and CQRS projections without repeating external side effects

---

# References (research trail)

**Primary documentation — Kotlin and coroutines**
- Kotlin — *Coroutines basics*: https://kotlinlang.org/docs/coroutines-basics.html
- Kotlin API — `Deferred<T>` (`join`, `await`, failure semantics): https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/
- Kotlin API — `Dispatchers.IO`: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-i-o.html
- Kotlin — *Asynchronous Flow*: https://kotlinlang.org/docs/flow.html

**Primary documentation — Spring Boot and Spring Framework**
- Spring Boot — *Kotlin support*: https://docs.spring.io/spring-boot/reference/features/kotlin.html
- Spring — *Building web applications with Spring Boot and Kotlin*: https://spring.io/guides/tutorials/spring-boot-kotlin/
- Spring Framework — *Coroutines*: https://docs.spring.io/spring-framework/reference/languages/kotlin/coroutines.html
- Spring MVC — *Asynchronous requests*: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html
- Spring Framework — *HTTP interface client*: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-interface
- Spring Framework — *Resilience*: https://docs.spring.io/spring-framework/reference/core/resilience.html
- Spring Framework — *Component scanning and stereotype annotations*: https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html
- Spring Framework — *Spring projects in Kotlin (`kotlin-spring`/all-open)*: https://docs.spring.io/spring-framework/reference/languages/kotlin/spring-projects-in.html
- Spring Boot — *Build systems and Boot 4's WebClient starter*: https://docs.spring.io/spring-boot/reference/using/build-systems.html
- Spring Data Relational — *Spring Data JDBC*: https://docs.spring.io/spring-data/relational/reference/jdbc.html
- Spring Data Relational — *Spring Data R2DBC*: https://docs.spring.io/spring-data/relational/reference/r2dbc.html
- Spring Framework — *Spring WebFlux*: https://docs.spring.io/spring-framework/reference/web/webflux.html
- Spring Modulith — *Application events and externalization*: https://docs.spring.io/spring-modulith/reference/events.html
- Spring Kafka — *Transactions*: https://docs.spring.io/spring-kafka/reference/kafka/transactions.html
- AWS Prescriptive Guidance — *Saga patterns*: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-patterns.html
- Azure Architecture Center — *Event-driven architecture*: https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven
- Azure Architecture Center — *Event sourcing*: https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing

**Secondary perspectives and comparison material**
- JetBrains — *Strategic partnership with Spring* (industry adoption and Kotlin/Spring direction): https://blog.jetbrains.com/kotlin/2025/05/strategic-partnership-with-spring/
- InfoQ — *The Spring Team on Spring Framework 7 and Spring Boot 4*: https://www.infoq.com/articles/spring-team-spring-7-boot-4/
- Arho Huttunen — *Hexagonal Architecture With Spring Boot*: https://www.arhohuttunen.com/hexagonal-architecture-spring-boot/
- codecentric — *Spring Modulith with Kotlin and hexagonal architecture*: https://www.codecentric.de/en/knowledge-hub/blog/modularization-the-easy-way-spring-modulith-with-kotlin-and-hexagonal-architecture
- Coroutine/Reactor/virtual-thread benchmark repository: https://github.com/gaplo917/coroutine-reactor-virtualthread-microbenchmark

Build it once without AI autocomplete, narrating your decisions out loud like it's the interview. 🚀
