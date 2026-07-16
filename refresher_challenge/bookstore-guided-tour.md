# 🚌 The Bookstore Build
## A Guided Tour of Modern Kotlin APIs — Spring Boot 4, Spring Framework 7, Hexagonal Architecture

You're about to build a small backend, twice as carefully as it strictly needs. That's the point. This is a **learning tour**: every stop teaches you one cluster of Kotlin/Spring skills, in the order that makes each lesson land, and every stop ends with something you can *say out loud* — because if you can't explain a decision in two sentences, you haven't finished learning it.

**How to travel:** read a stop, build it, run its tests, then say the "🎤 Say it out loud" line without looking. Don't skip ahead — the order is the curriculum. Budget a few evenings.

---

## 🏪 The story before we leave

Imagine a small bookstore that sells through a web storefront, an in-store terminal, and phone orders. Their nightmare is simple and expensive: **two customers buy the last copy of the same book at the same instant.** One of them will be disappointed either way — the question is whether the *system* decides cleanly (one succeeds, one gets a polite "sold out"), or whether both orders go through and someone gets an apologetic email and a refund.

Fixing this properly turns out to require almost everything that matters in backend engineering:

- **Money and identity have rules.** Prices are `BigDecimal` (never floating point), ISBNs are unique, stock can't go negative, and an order's total must equal the sum of its lines *at the moment of purchase*. These are business invariants, and they must hold no matter which client calls.
- **Concurrency is real.** The last-copy race is a transaction-and-locking problem, not a UI problem.
- **Clients come and go, the contract stays.** A mobile app today, a partner marketplace tomorrow. The rules must live in one place, behind a versioned HTTP contract.
- **Someone runs this at 2 a.m.** They need to trace one failing request through the logs, check health, and read errors a machine can parse.

So you'll build the **Bookstore API**: the single system of record for the catalog and order intake. Two aggregates only — `Book` and `Order` — because that's the smallest domain that still forces you through relations, transactions, locking, validation, external calls, and versioning. Small surface, production-grade depth.

**What we're deliberately NOT building:** payments (marking an order `PAID` is a status flip, not a Stripe integration), auth, Kafka, microservices, caching-as-correctness, any frontend. Cutting these is what keeps the project honest.

---

## 🗺️ The map: how the tour is laid out

Most Spring tutorials build outside-in: controller, then service, then repository. You end up with business rules smeared across all three and a domain shaped like your database tables.

We're going **inside-out**, following **hexagonal architecture** (a.k.a. ports & adapters). If you remember one thing from the whole tour, make it this:

> **The one rule: dependencies point inward.** The business core knows nothing about HTTP, JPA, or Spring. Everything technical is an *adapter* plugged into a *port* — a plain Kotlin interface owned by the core.

```
            ┌──────────────────────────────────────────────────┐
  driving   │                    adapters                      │  driven
  (input)   │  ┌────────────────────────────────────────────┐  │  (output)
            │  │              application                   │  │
 REST ──────┼─▶│  input ports   ┌──────────┐  output ports  │◀─┼────── JPA/Postgres
 controller │  │  (use cases)   │  domain  │  (interfaces)  │  │
            │  │                │ Book,    │                │◀─┼────── HTTP client →
 (someday:  │  │  PlaceOrder    │ Order,   │ BookRepository │  │       price service
  GraphQL,  │  │  ManageBooks   │ rules    │ PriceGateway   │  │
  CLI, MQ)  │  └────────────────└──────────┘────────────────┘  │
            └──────────────────────────────────────────────────┘
                      every arrow of knowledge points inward
```

Why bother, on a project this small? Because hexagonal has a real cost — mapping boilerplate, more files, discipline — and a real payoff — swappable technology, business logic you can test in milliseconds without booting Spring, decisions you can defer. On a big messy codebase you can't *see* the pattern anymore. On this one, you will literally compile and test your business rules **before Spring Web or JPA exist in the project**. That experience is the lesson.

Your itinerary, and what each stop refreshes:

| Stop | You build | You learn |
|---|---|---|
| 0 | Walking skeleton | Boot 4 setup, Docker Compose, Flyway, typed config |
| 1 | The domain | Kotlin class design, invariants, framework-free code |
| 2 | Ports & use cases | Interfaces as contracts, DIP, constructor injection |
| 3 | Persistence adapter | The Kotlin/JPA minefield, Testcontainers |
| 4 | Web adapter | Controllers, DTOs, validation, the `@field:` trap |
| 5 | Error handling | `@RestControllerAdvice`, RFC 9457 ProblemDetail |
| 6 | Logging | SLF4J idioms, MDC correlation IDs |
| 7 | Pagination & filtering | `Pageable`, derived queries, JPQL |
| 8 | Order placement | `@Transactional`, optimistic locking, the last-copy race |
| 9 | The test pyramid | MockK, slice tests, Testcontainers, ArchUnit |
| 10 | The 2026 wing | Spring 7 API versioning, HTTP interface clients, `@Retryable`, virtual threads |

Three **detours** along the way tackle the questions that don't have annotation-shaped answers: *should every layer get an interface?* (after Stop 2), *coroutines or virtual threads?* and *Retrofit or RestClient?* (both at Stop 10, where you'll have the context to care).

---

## 🎒 Packing list (do this before Stop 0)

**The stack, and why each piece (current as of mid-2026):**

- **Spring Boot 4.x** on **Spring Framework 7** — the new baseline generation: Kotlin 2.2+, Java 17+ (use **21 or newer** so virtual threads are available), Jakarta EE 11. Boot 3.5 hit open-source EOL in June 2026, so 4.x is simply where new projects start.
- **Kotlin 2.2+** — K2 compiler, and (this matters, remember it for Stop 10) Spring's APIs are now annotated with **JSpecify** nullness annotations that Kotlin translates into *real* nullable types instead of platform types.
- **Gradle Kotlin DSL** — what most Kotlin shops use.
- **PostgreSQL 16** via Docker Compose, **Flyway** for migrations, **JUnit 5 + MockK + Testcontainers + ArchUnit** for tests.

Go to [start.spring.io](https://start.spring.io) → Kotlin, Gradle–Kotlin → add: **Spring Web, Spring Data JPA, Validation, PostgreSQL Driver, Flyway Migration, Actuator, Testcontainers, Docker Compose Support**.

**Read your own `build.gradle.kts` plugins block like an interviewer will ask about it — because they will:**

```kotlin
plugins {
    id("org.springframework.boot") version "4.0.x"
    id("io.spring.dependency-management") version "1.1.x"
    kotlin("jvm") version "2.2.x"
    kotlin("plugin.spring") version "2.2.x"  // ⭐ memorize why
    kotlin("plugin.jpa") version "2.2.x"     // ⭐ memorize why
}

kotlin {
    compilerOptions { freeCompilerArgs.add("-Xjsr305=strict") }
}
```

- **`plugin.spring` (all-open):** Kotlin classes are `final` by default, but Spring creates CGLIB proxies *by subclassing* for `@Transactional`, `@Configuration`, `@Async`… This plugin opens Spring-annotated classes automatically. Without it: startup failures, or worse, methods that are silently non-transactional.
- **`plugin.jpa` (no-arg):** Hibernate instantiates entities reflectively through a **no-arg constructor**, which Kotlin constructor-property classes don't have. The plugin synthesizes one for `@Entity`/`@Embeddable`/`@MappedSuperclass`.

**`compose.yaml`** at the project root — Boot auto-detects and starts it:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: bookstore
      POSTGRES_USER: bookstore
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
```

**The schema** you'll create as Flyway migrations (`V1__create_books.sql`, `V2__create_orders.sql`) when you reach Stops 3 and 8:

```
books:        id (bigserial PK), isbn (unique, not null), title, author,
              price NUMERIC(10,2), stock int, version bigint, created_at, updated_at
orders:       id, customer_email, status (CREATED|PAID|CANCELLED), total, created_at
order_items:  id, order_id (FK), book_id (FK), quantity, unit_price
```

Two habits from day one: money is `NUMERIC(10,2)` ↔ `BigDecimal`, **never** `Double`; and that `version` column is the optimistic-locking hero of Stop 8.

---

## 🚏 Stop 0 — The walking skeleton

**Goal:** an app that boots, migrates, and reads typed configuration. Nothing else.

1. Start the app; Boot spins up Compose Postgres; Flyway runs `V1`; set `spring.jpa.hibernate.ddl-auto=validate` right now and never change it. Flyway owns the schema; Hibernate only checks that the mapping matches. ("I never let Hibernate generate schema in prod" is a free interview point — `ddl-auto=update` in production is how columns mysteriously appear and data mysteriously dies.)
2. Add a business setting to `application.yml` — `bookstore.orders.max-items-per-order: 10` — and bind it to an **immutable** class:

```kotlin
@ConfigurationProperties(prefix = "bookstore.orders")
data class OrderProperties(val maxItemsPerOrder: Int = 10)
```

3. Enable with `@ConfigurationPropertiesScan` on the application class. Add an `application-local.yml` profile and run with it, just to touch profiles once.

**🎤 Say it out loud:** "Constructor-bound `@ConfigurationProperties` data classes beat `@Value`: type-safe, immutable, validatable with jakarta annotations, and discoverable through configuration metadata."

---

## 🚏 Stop 1 — Build the heart first: the domain (no Spring allowed)

**Goal:** the business rules of the bookstore, in pure Kotlin. If you `import org.springframework` or `import jakarta.persistence` anywhere in this package, you've taken a wrong turn.

This is the stop most tutorials never make, and it's where Kotlin itself is the curriculum: classes vs data classes, `init` blocks, `require`, sealed thinking, value semantics.

1. Create `domain/model`:
   - `Book` — enforces its own invariants (`init { require(price > BigDecimal.ZERO) }`, stock ≥ 0) and owns the behavior that matters: `fun decreaseStock(quantity: Int)` which throws `InsufficientStockException` rather than letting anyone set stock to -3.
   - `Order` + `OrderItem` — the total is *computed from the items*, never accepted from outside. An order can't exist with zero items.
   - `OrderStatus` — a Kotlin `enum class`: `CREATED, PAID, CANCELLED`.
2. Create `domain/error`: `BookNotFoundException`, `DuplicateIsbnException`, `InsufficientStockException` — small, specific, meaningful.
3. Write **plain JUnit tests**. No Spring context, no mocks, no annotations beyond `@Test`. Order math, stock rules, invariant violations. They run in milliseconds.

**What just happened:** you wrote the most important code in the system and Spring wasn't invited. This package would compile unchanged inside a Ktor app, a CLI tool, or a Lambda. *That* is the hexagon's core, demonstrated rather than described.

**🎤 Say it out loud:** "My domain layer has zero framework imports. I can test every business rule without booting a container, and I have an architecture test that keeps it that way."

---

## 🚏 Stop 2 — Draw the doors: ports and use cases

**Goal:** define *what the application does* (input ports) and *what it needs from the world* (output ports), then implement the use cases against those contracts — still without a database or a controller in sight.

1. `application/port/in` — the use cases, named so the system's capabilities are greppable:
   - `ManageBooksUseCase`, `QueryBooksUseCase`, `PlaceOrderUseCase`.
2. `application/port/out` — what the core needs:
   - `BookRepositoryPort`, `OrderRepositoryPort`, `PriceCheckPort` (one method: verify a price — this becomes an HTTP call at Stop 10, but the core will never know that).
   - Kotlin detail with interview mileage: return **nullable types, not `Optional`** — `fun findByIsbn(isbn: String): Book?`. Idiomatic, and Spring Data understands it natively.
3. `application/service` — `BookService`, `PlaceOrderService` implement the input ports, depending only on the output ports via **constructor injection of `val`s**. Enforce `maxItemsPerOrder` from Stop 0 here.
4. Test with **MockK**, mocking the *ports*: the duplicate-ISBN rejection, the item-limit breach — still no Spring context.

```kotlin
val repo = mockk<BookRepositoryPort>()
val service = BookService(repo)

@Test
fun `throws when isbn already exists`() {
    every { repo.findByIsbn("123") } returns existingBook
    assertThrows<DuplicateIsbnException> { service.create(commandWith("123")) }
    verify(exactly = 0) { repo.save(any()) }
}
```

(Backtick function names are idiomatic Kotlin test style — use them.)

**🎤 Say it out loud:** "Constructor injection over field injection: the dependency is immutable, the object can't exist in an invalid state, tests need no reflection, and circular dependencies fail fast. In Kotlin, field injection would also force `lateinit var` — surrendering null-safety for nothing."

### 🔀 Detour A — "Wait, should *every* layer get an interface?"

You've just written a bunch of interfaces, so let's settle this before the habit spreads. Two very different practices hide under "interfaces between layers":

**1. Ports — interfaces at architectural boundaries. Always yes.**
`BookRepositoryPort` is owned by the application layer; the JPA adapter (Stop 3) will *conform to it*. This is the Dependency Inversion Principle doing real work: high-level policy defines the contract, low-level detail implements it, and in tests you swap in a fake in one line. Same for `PriceCheckPort` — at Stop 10 you'll swap what's behind it without the core noticing, and that moment is the architecture proving its worth.

**2. The `FooService` / `FooServiceImpl` reflex — an interface for every class, single implementation, same package. No.**
This ritual came from the EJB era. Today: Spring proxies concrete classes via CGLIB (that's literally why you added `plugin.spring`), MockK mocks final classes without complaint, and a one-implementation interface in the same package is speculative generality — pure ceremony. Add an interface *within* a layer only when a second implementation or a genuine boundary exists.

**What about the input ports you just wrote?** Honest answer: they're the most debatable interfaces in the project. Controller → service already points inward, so many production teams let controllers call the service class directly and skip `PlaceOrderUseCase` entirely. We keep them here for the learning value (the system's capabilities read like a table of contents) — but know that dropping *input* ports is a defensible trim, while dropping *output* ports collapses the whole architecture.

**🎤 Say it out loud:** "DIP is about the direction of the dependency arrow, not the `interface` keyword. I put interfaces where the arrow needs to flip — at the ports — and use concrete classes everywhere else."

**A quick SOLID checkpoint while we're parked here** — you've now touched all five, in code rather than on a slide:
- **S**: controllers translate HTTP, services orchestrate one operation each, adapters translate technology — a change in JSON shape, business rule, or SQL touches exactly one place.
- **O**: a new delivery mechanism or database is a new adapter; the core doesn't change.
- **L**: every implementation of a port must honor its contract — a test fake whose `findByIsbn` throws instead of returning `null` poisons every test built on it.
- **I**: ports are small and role-shaped (`PriceCheckPort` has one method); no adapter stubs methods it doesn't serve.
- **D**: `PlaceOrderService` depends on abstractions *it owns*; JPA and HTTP details depend on those abstractions.

---

## 🚏 Stop 3 — The database room: the persistence adapter (a.k.a. the Kotlin/JPA minefield)

**Goal:** give the ports a real Postgres implementation — and survive the most-asked Kotlin+Spring interview territory on the way.

1. Write `BookJpaEntity` in `adapter/out/persistence`. **Deliberately not a `data class`** — a normal class with `var`s:

```kotlin
@Entity
@Table(name = "books")
class BookJpaEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,

    @Column(nullable = false, unique = true)
    var isbn: String,

    var title: String,
    var author: String,
    var price: BigDecimal,
    var stock: Int,

    @Version
    var version: Long = 0,
)
```

**Why no data class? (top-3 Kotlin+Spring interview question — learn it cold):**
- Generated `equals`/`hashCode` use *all* properties, including mutable ones and the DB-generated id → identity breaks across the persistence lifecycle (put an entity in a `HashSet` before save, look for it after — gone).
- `toString` walks associations → can trigger lazy loading or infinite recursion.
- `copy()` invites accidental detached duplicates.
- Rule of thumb: **data classes for DTOs, always. For entities, plain class with stable id-based equality.**

2. Auditing: add `createdAt`/`updatedAt` with `@CreatedDate`/`@LastModifiedDate`, `@EntityListeners(AuditingEntityListener::class)`, and `@EnableJpaAuditing` in config.
3. The Spring Data repository stays **`internal` to the adapter package** — it's an implementation detail:

```kotlin
internal interface SpringDataBookRepository : JpaRepository<BookJpaEntity, Long> {
    fun findByIsbn(isbn: String): BookJpaEntity?   // nullable, not Optional
}
```

4. Now the piece that makes this hexagonal: `BookPersistenceAdapter` **implements `BookRepositoryPort`** and maps between worlds with extension functions — idiomatic Kotlin mapping:

```kotlin
fun BookJpaEntity.toDomain(): Book = ...
fun Book.toEntity(): BookJpaEntity = ...
```

Yes, this mapping is boilerplate. That's the tuition. The alternative — putting `@Entity` on your domain model — is the pragmatic shortcut many teams take, and now you know exactly what it costs them: the ORM starts dictating the domain's design (mutable `var`s, nullable ids, open classes, no-arg constructors), and "persistence-ignorant business logic" quietly dies. You've earned the right to have an opinion either way.

5. Test the adapter with `@DataJpaTest` + **Testcontainers** Postgres via `@ServiceConnection` — one annotation, zero manual datasource config, and it's a *real* Postgres. (H2-vs-real-database behavioral differences are exactly why Testcontainers is the modern answer.)

**🎤 Say it out loud:** "My JPA entities never leave the persistence adapter. The core sees domain objects through a port; the ORM is an implementation detail I could swap."

---

## 🚏 Stop 4 — The front door: REST controllers, DTOs, validation

**Goal:** expose Book CRUD over HTTP without ever letting an entity or domain object touch JSON.

The endpoints:

```
POST   /api/v1/books          → 201 + Location header
GET    /api/v1/books/{id}     → 200 / 404
PUT    /api/v1/books/{id}     → 200 / 404
DELETE /api/v1/books/{id}     → 204 / 404
GET    /api/v1/books          → 200 (paged — Stop 7)
```

1. **The DTO boundary is absolute.** `CreateBookRequest`, `UpdateBookRequest`, `BookResponse` as data classes (here they're *perfect* — value semantics, `copy()` for test builders). Map with an extension function: `fun Book.toResponse(): BookResponse`.
2. Validate the request:

```kotlin
data class CreateBookRequest(
    @field:NotBlank @field:Size(min = 10, max = 17)
    val isbn: String,
    @field:NotBlank val title: String,
    @field:NotBlank val author: String,
    @field:Positive val price: BigDecimal,
    @field:PositiveOrZero val stock: Int,
)
```

**⚠️ The trap everyone hits once:** that `@field:` use-site target. Without it, Kotlin puts `@NotBlank` on the constructor **parameter**, where Bean Validation may never look — and validation *silently does nothing*. Boot 4 improves the defaults here, but knowing *why* `@field:` exists is the interview answer.

3. Keep the controller thin: translate HTTP ↔ use-case calls, nothing more. It depends on the **input port**, not the service class. Return `ResponseEntity` where you need control (201 + `Location`), plain values elsewhere.

**🎤 Say it out loud:** "Controllers never expose entities. Request and response DTOs are the API contract; the domain model is free to change without breaking clients."

---

## 🚏 Stop 5 — When things go wrong: one place, one format

**Goal:** every non-2xx response in the whole API is an **RFC 9457 `ProblemDetail`**, produced by a single `@RestControllerAdvice`.

Map, in one advice class:
- `BookNotFoundException` → **404**
- `DuplicateIsbnException` → **409**
- `MethodArgumentNotValidException` → **400**, with a field-error map tucked into `ProblemDetail.properties`
- `ObjectOptimisticLockingFailureException` → **409** (you'll trigger this for real at Stop 8)
- fallback `Exception` → **500** with a generic message — never leak stack traces

Then *test the JSON shape*: POST a duplicate ISBN and an invalid body; assert `type`, `title`, `status`, `detail`.

**🎤 Say it out loud:** "Spring gives me a standards-based error contract for free with `ProblemDetail` — RFC 9457, which supersedes 7807. One advice class, machine-readable errors everywhere."

---

## 🚏 Stop 6 — Leaving footprints: logging that helps at 2 a.m.

**Goal:** logs you could actually debug production with.

1. Pick ONE logger idiom and be ready to defend it:

```kotlin
// Option A: companion object
companion object { private val log = LoggerFactory.getLogger(BookService::class.java) }

// Option B: a reified top-level helper you write once
inline fun <reified T> T.logger(): Logger = LoggerFactory.getLogger(T::class.java)
```

2. Right levels: `INFO` for business events ("Order 42 placed, total=59.90"), `DEBUG` for flow detail, `WARN` for recoverable oddities, `ERROR` only with a stack trace and only where handled.
3. **Parameterized messages** — `log.info("Book created id={}", id)` — never Kotlin interpolation `"$id"` in log calls. Interpolation is evaluated even when the level is off; `{}` isn't.
4. The showpiece: a servlet `OncePerRequestFilter` that generates a `requestId`, puts it in **MDC**, and clears it after. Add `%X{requestId}` to the log pattern. Now every line of a request is traceable — this is the canonical answer to "how would you trace one request across the logs?"
5. `logging.level.org.hibernate.SQL=debug` in the local profile only — and know why `show-sql=true` is worse (it prints to stdout, bypassing the logging system entirely).

---

## 🚏 Stop 7 — Browsing the shelves: pagination, sorting, filtering

**Goal:** `GET /api/v1/books?page=0&size=20&sort=title,asc&author=tolkien&maxPrice=30`.

1. Accept a `Pageable` in the controller, return `Page<BookResponse>` via `page.map { it.toResponse() }`.
2. Do both filter styles once each, so you've refreshed both: a derived query (`findByAuthorContainingIgnoreCase`) and a JPQL `@Query` with `@Param`.
3. **Cap the page size** (say, 100). The reason has a name worth dropping: pagination-DoS — `?size=100000` should not be a load test a stranger can run against you.

**A hexagonal wrinkle worth noticing:** if the *input port* takes Spring's `Pageable`, the framework just leaked into your core. The strict fix is a tiny `PageQuery`/`PagedResult` pair of your own (~5 lines each) that the web adapter translates into `Pageable`. Do it the strict way here to feel the cost — and understand why plenty of teams shrug and accept the leak on query-only paths. Both positions are defensible *if you can argue them*; that's the actual lesson.

**🎤 Quick self-quiz:** `Page` vs `Slice`? (`Page` runs an extra `count` query to know the total; `Slice` only knows if there's a next page.)

---

## 🚏 Stop 8 — The main event: orders, transactions, and the last-copy race

**Goal:** the reason the bookstore hired you. `POST /api/v1/orders` with `{ customerEmail, items: [{bookId, quantity}] }`, correct under concurrency.

1. `PlaceOrderService.place()` does the whole dance **in a single `@Transactional` method**: load the books → verify prices through `PriceCheckPort` → check stock → decrement stock → compute total server-side (client totals are never trusted) → persist order + items. Any failure anywhere → everything rolls back.
2. Business rules → business exceptions → your Stop 5 advice: `InsufficientStockException` → 409; more than `maxItemsPerOrder` items (from Stop 0's properties — see how it all connects?) → 400.
3. `status` with `@Enumerated(EnumType.STRING)` — and know why `ORDINAL` is a landmine: reorder the enum constants once and you've silently corrupted every historical row.
4. **The race, resolved:** that `@Version` column from the packing list means two concurrent orders for the last copy end with one commit and one `ObjectOptimisticLockingFailureException` → 409. **Prove it with a two-thread test.** Watching one thread win and one get a clean 409 is the most satisfying moment of the tour.
5. `GET /api/v1/orders/{id}` returns the order with items — and here's where `LazyInitializationException` ambushes people: map to DTO *inside* the transaction, or use a fetch join. Know both fixes (fetch join, `@EntityGraph`) for the related N+1 question.

**A small confession about purity:** `@Transactional` is a Spring annotation sitting in your application layer, which was supposed to be framework-light. Purists hide it behind a decorator or port; pragmatists annotate the use case and move on. We annotate — it's declarative metadata, not control flow, and the ceremony to hide it teaches less than it costs. The mature position isn't "never leak" — it's "know exactly where you leaked, and why."

**🎤 Say these out loud — Stop 8 is dense with interview gold:**
- "`@Transactional` works through a proxy, so **self-invocation bypasses it** — `this.otherTransactionalMethod()` never crosses the proxy boundary."
- "Default rollback is unchecked exceptions only. Kotlin has no checked exceptions, but Spring applies the Java rule *by exception type* — a Kotlin function throwing `IOException` won't roll back unless I set `rollbackFor`."
- "`readOnly = true` on query methods skips dirty checking and can hint driver/replica routing."
- "Optimistic locking for low contention — detect the conflict, fail one side, let it retry. Pessimistic for hot rows where retries would be constant."

---

## 🚏 Stop 9 — Prove it: the test pyramid, plus a test for the architecture itself

**Goal:** every ring of the system tested with the cheapest tool that can prove it — and the hexagon's rules enforced by a machine, not by good intentions.

Walk up the pyramid:

1. **Domain** — plain JUnit (done at Stop 1). No Spring, no mocks, milliseconds.
2. **Application** — MockK against the ports (done at Stop 2; extend for order placement: the last-copy scenario, the limit breach). Know `every` / `verify` / `slot`.
3. **Web adapter** — `@WebMvcTest(BookController::class)` with a mocked input port: assert status codes, JSON bodies, the validation 400 path, and the ProblemDetail shape.
4. **Persistence adapter** — `@DataJpaTest` + Testcontainers `@ServiceConnection` (done at Stop 3).
5. **The whole hexagon** — one `@SpringBootTest` + Testcontainers walking the full place-order flow, including the rollback and the optimistic-lock 409.
6. **The architecture itself** — ArchUnit. This is the test that keeps Stop 1's promise forever:

```kotlin
@AnalyzeClasses(packages = ["com.example.bookstore"])
class HexagonalRulesTest {
    @ArchTest
    val domainIsPure = noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "org.springframework..", "jakarta.persistence..", "..adapter..")

    @ArchTest
    val adaptersDontKnowEachOther = noClasses().that().resideInAPackage("..adapter.in..")
        .should().dependOnClassesThat().resideInAPackage("..adapter.out..")
}
```

(By the way, this is also why we could keep the project a single Gradle module instead of splitting core/api/data into separate modules: ArchUnit gives us the boundary enforcement without the build ceremony. If this ever grows into a team codebase, promoting packages to modules is mechanical — precisely because the dependencies already point the right way.)

**🎤 Say it out loud:** "MockK over Mockito for Kotlin: it handles final classes, `suspend` functions, and extension functions, with a Kotlin DSL. And my architecture isn't a diagram on a wiki — it's a failing test if someone breaks it."

---

## 🚏 Stop 10 — The 2026 wing: what's actually new in Spring 7 / Boot 4

**Goal:** touch the headline features of the new generation, so your refresher is current and your project is differentiated. Four exhibits and two detours.

### Exhibit 1 — First-class API versioning

Until Framework 7, versioning was DIY: URL prefixes, custom header parsing, duplicated controllers. Now it's native: you configure a strategy once, then declare versions right on the mappings.

Keep `/api/v1` as your coarse contract, and demonstrate the new machinery on one endpoint:

```kotlin
@Configuration
class WebConfig : WebMvcConfigurer {
    override fun configureApiVersioning(configurer: ApiVersionConfigurer) {
        configurer.useRequestHeader("X-API-Version")
    }
}

@GetMapping("/{id}", version = "1.0")
fun getBookV1(@PathVariable id: Long): BookResponse = ...

@GetMapping("/{id}", version = "1.1")   // v1.1 adds a field
fun getBookV11(@PathVariable id: Long): BookResponseV11 = ...
```

Path, header, query-param, and media-type strategies are all supported, with built-in deprecation signaling (RFC 9745) to warn clients before you sunset a version. Cheap to build, very current, and it makes the "how do you evolve an API without breaking clients?" conversation concrete.

### Exhibit 2 — The outbound call: an HTTP interface client for the price check

Remember `PriceCheckPort` from Stop 2? Time to put a real HTTP call behind it — and this is where your "Retrofit or something else?" question gets its answer.

Declare the client as an annotated interface, and let Boot 4 wire it with a single annotation:

```kotlin
// adapter/out/pricing
interface PriceCheckClient {
    @GetExchange("/prices/{isbn}")
    fun currentPrice(@PathVariable isbn: String): PriceQuote
}

@Configuration
@ImportHttpServices(PriceCheckClient::class)   // Boot 4: that's the whole wiring
class HttpClientsConfig
```

Then `PriceCheckAdapter` implements the port and delegates to the client. The application core still only knows `PriceCheckPort` — you just plugged a technology into a socket, which is the hexagon paying rent again.

### 🔀 Detour B — Retrofit, RestTemplate, WebClient, RestClient… which one, really?

The elimination round, as of 2026:

- **`RestTemplate`** — in maintenance mode, feature-frozen. Legacy code only; don't start anything new on it.
- **`WebClient`** — excellent *if* you're on the reactive stack. In a blocking servlet app it drags the entire Reactor machinery (`Mono`, `Flux`, Netty) along for zero benefit.
- **Retrofit** — a genuinely great library, and the obvious *inspiration* for Spring's HTTP interfaces (both are "annotated interface → generated client"). But in a Spring Boot server it's a guest in someone else's house: you hand-wire OkHttp, converters and factories, carry an extra dependency surface, and forfeit native integration. It remains the right answer **on Android**. On the server, the framework caught up.
- **`RestClient`** — the modern default: `WebClient`'s fluent API on the plain blocking servlet stack, no reactive overhead, and it plays perfectly with virtual threads (wait for Detour C).
- **HTTP interface clients** — the declarative layer *on top of* `RestClient`: Retrofit ergonomics, natively. In Spring 7 they even participate in API versioning (the client can inject the version via `ApiVersionInserter` configuration, so calling code never fiddles with headers) and group-level configuration for multi-service setups.

**🎤 Say it out loud:** "For a new blocking Spring service: declarative HTTP interfaces backed by `RestClient`. Retrofit ergonomics, zero extra dependencies, virtual-thread-friendly. `WebClient` only if I'm genuinely reactive; `RestTemplate` only in code I inherited."

### Exhibit 3 — Resilience moved into the framework

Spring 7 absorbed the core of Spring Retry: annotate the flaky edge, declaratively.

```kotlin
@Retryable(maxAttempts = 3, delay = 200, jitter = 100)  // backoff + jitter, in-framework
override fun verifyPrice(isbn: String): PriceQuote = client.currentPrice(isbn)
```

Put it on the price-check adapter (the *edge*, never inside the domain). There's a sibling, `@ConcurrencyLimit`, that acts as a declarative bulkhead — hold that thought for about thirty seconds.

### Exhibit 4 — Flip one switch: virtual threads

In `application.yml`:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Done. Every request handler, every `RestClient` call, every repository call now runs on a virtual thread — blocking stops costing an OS thread per request, and the thread-per-request model scales to tens of thousands of concurrent requests with **zero code changes**.

### 🔀 Detour C — Coroutines vs virtual threads: the honest version

This is *the* 2026 concurrency interview topic, so let's get you an opinion you can defend.

**The core distinction:** suspending is a property of your *code* (`suspend` functions, explicit pause points, a compiler transformation); virtual threads are a property of the *runtime* (the JVM makes blocking cheap, invisibly). Kotlin gives you both — so the question is never "which is better," it's "which problem do I have?"

**Why this app defaults to virtual threads:**
- Its hot path is **JPA/JDBC — a blocking API**. Coroutines don't make JDBC non-blocking; they'd just relocate the blocking onto another dispatcher. Virtual threads make the blocking itself nearly free, without `suspend` coloring your entire call stack.
- One property, no rewrite, and the code stays the boring sequential style everyone can read.
- The new footgun to know: virtual threads make *threads* cheap, but your Postgres connection pool is still finite. Unbounded cheap concurrency stampeding a pool of 10 connections is why `@ConcurrencyLimit` exists — bulkheads matter *more* with virtual threads, not less. (Drop that in an interview and watch eyebrows rise.)

**Where coroutines genuinely win — and where you'd flip the decision:**
- **Structured concurrency.** Fan out three remote calls, and if one fails, the siblings are cancelled automatically:

```kotlin
suspend fun enrich(isbn: String): Enriched = coroutineScope {
    val price = async { priceClient.currentPrice(isbn) }
    val rating = async { ratingClient.rating(isbn) }
    Enriched(price.await(), rating.await())   // one fails → the other is cancelled
}
```

  Every coroutine has a parent, lifetimes are bounded, leaks are prevented — virtual threads only match this with explicit `StructuredTaskScope`, which is newer and clunkier.
- **A genuinely reactive stack** (WebFlux + R2DBC): coroutines and `Flow` are the idiomatic, readable face of it.
- And they compose: coroutines can dispatch *onto* virtual threads (`Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()`). It's not a war; it's a toolbox.

**Your exercise for this detour:** convert the price-check orchestration to a `suspend` flow with `coroutineScope` fan-out (Spring MVC handles `suspend` controller functions fine), run both versions, and *feel* the difference in code shape. Then delete whichever version you like less — but keep the argument.

**🎤 Say it out loud:** "My JPA app blocks either way, so I let the runtime absorb it: virtual threads, one property, no rewrite. I reach for coroutines the moment I'm composing multiple remote calls and need structured cancellation — or if I were on R2DBC. And since virtual threads make concurrency cheap, I bulkhead scarce resources like connection pools with `@ConcurrencyLimit`."

### One more 2026 line for the road

Boot 4 / Framework 7 finished migrating the whole portfolio to **JSpecify** null-safety annotations, and Kotlin translates them into real Kotlin nullability. Practically: `repository.findByEmail(...)` comes back as `User?` — an honest nullable type the compiler enforces — instead of a platform type that explodes at runtime. Keep `-Xjsr305=strict` for the older third-party libraries that haven't migrated. This is the single best "why Boot 4 matters *specifically for Kotlin*" line you can drop.

---

## 🌶️ Side quests (pick any, 30–60 min each)

- **Actuator hardening:** expose only `health,info,metrics`; write a custom `HealthIndicator` that checks book count > 0.
- **OpenAPI:** add `springdoc-openapi` → free Swagger UI; annotate one endpoint completely.
- **Caching:** `@EnableCaching` + `@Cacheable`/`@CacheEvict` on book reads — then explain why caching mutable *stock* is dangerous (staleness on the field that must never lie).
- **Spring Modulith:** verify module boundaries as a complement to ArchUnit — it pairs naturally with hexagonal packages.
- **Kotlin Serialization:** Boot 4 ships a `spring-boot-kotlin-serialization` starter; swap Jackson on one endpoint and compare ergonomics.

---

## 🏁 You made it if…

- [ ] The app boots via Docker Compose Postgres; Flyway migrates; `ddl-auto=validate`
- [ ] The `domain` package has **zero** Spring/JPA imports — and an ArchUnit test proves it
- [ ] Domain + application layers are fully tested without a Spring context
- [ ] Full Book CRUD with DTOs, validation, correct status codes (201/204/404/409)
- [ ] Every error is an RFC 9457 `ProblemDetail` from one advice class
- [ ] Every log line carries the MDC `requestId`
- [ ] Paged + filtered listing, size capped
- [ ] Transactional order placement; a two-thread test proves the optimistic-lock 409
- [ ] The price check runs through an HTTP interface client behind a port, with `@Retryable`
- [ ] One endpoint is served in two versions via Spring 7's native API versioning
- [ ] Virtual threads are on, and you can argue Detour C for two minutes unprompted
- [ ] You can answer everything in the souvenir shop below without looking

---

## 🎁 Souvenir shop: take these with you

### The annotation cheat sheet (cover the right column, quiz yourself)

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
| `@Retryable` / `@ConcurrencyLimit` | Framework 7 in-core resilience (retry with backoff; bulkhead) |
| `@GetMapping(version = "1.1")` | Framework 7 first-class API versioning |

### The rapid-fire quiz (answer out loud while building)

**Kotlin + Spring fundamentals:**
1. Why does Kotlin need `plugin.spring` and `plugin.jpa`? What breaks without each?
2. Why not data classes for JPA entities? Where *should* you use them?
3. Constructor vs field injection — three reasons, plus the Kotlin-specific one.
4. What does `@field:` do, and what silently fails without it?
5. How does `@Transactional` actually work? What's the self-invocation pitfall?
6. Optimistic vs pessimistic locking — when each, and how `@Version` behaves on conflict.
7. Why Flyway + `ddl-auto=validate` instead of `ddl-auto=update`?
8. `Page` vs `Slice` — what extra SQL does `Page` run?
9. How do you trace one request across log lines?
10. N+1 queries: how do you detect them; name two fixes.
11. Platform types: what do `-Xjsr305=strict` and JSpecify change in Boot 4?
12. MockK vs Mockito for Kotlin — why?

**Architecture & modern-stack round:**
13. State the hexagonal dependency rule in one sentence. How do you *enforce* it?
14. Input ports vs output ports — which would you drop under time pressure, and why?
15. Why is `FooServiceImpl` with a single implementation an anti-pattern in Spring/Kotlin?
16. Your domain model and JPA entities are separate classes — defend the mapping cost.
17. Where does `@Transactional` live in a hexagonal app, and what's the purist objection?
18. How would you add a GraphQL API to this system? What changes? (One new inbound adapter; nothing else.)
19. Retrofit vs Spring HTTP interfaces on the server — what changed in Framework 6/7?
20. `RestClient` vs `RestTemplate` vs `WebClient` — which for a new blocking MVC app?
21. Coroutines vs virtual threads for a Spring MVC + JPA service — and when do bulkheads matter *more*?
22. Spring's `Pageable` at the use-case boundary: leak or pragmatism? Argue both sides.
23. What's actually new in Spring Framework 7 / Boot 4? (API versioning, JSpecify null-safety, in-framework resilience, HTTP service registry, Jackson 3, `RestTestClient`, modularized starters.)

---

## 📚 Further reading (the research behind this tour)

**Spring Framework 7 / Boot 4:**
- InfoQ — *Spring Framework 7 and Spring Boot 4 Deliver API Versioning, Resilience, and Null-Safe Annotations*: https://www.infoq.com/news/2025/11/spring-7-spring-boot-4/
- InfoQ — *The Spring Team on Spring Framework 7 and Spring Boot 4*: https://www.infoq.com/articles/spring-team-spring-7-boot-4/
- Spring Framework 7.0 Release Notes: https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes
- Dan Vega — *What's New in Spring Framework 7 and Spring Boot 4*: https://www.danvega.dev/blog/spring-boot-4-is-here
- Baeldung — *Spring Boot 4 & Spring Framework 7 – What's New*: https://www.baeldung.com/spring-boot-4-spring-framework-7

**Hexagonal architecture with Spring + Kotlin:**
- Arho Huttunen — *Hexagonal Architecture With Spring Boot* (the JPA tension, pagination-through-ports discussion): https://www.arhohuttunen.com/hexagonal-architecture-spring-boot/
- Hieu Nguyen — *Hexagonal Architecture in Practice: Spring Boot + Kotlin*: https://medium.com/@hieunv/understanding-hexagonal-architecture-through-a-practical-application-2f2d28f604d9
- Gaurav Kumar — *Hexagonal Architecture in Kotlin Spring: A Quick Guide*: https://medium.com/@slashgkr/hexagonal-architecture-in-kotlin-spring-a-quick-guide-a5ecdc3c4c95
- Reference repo (multi-module, ArchUnit-enforced): https://github.com/dustinsand/hex-arch-kotlin-spring-boot
- codecentric — *Spring Modulith with Kotlin and hexagonal architecture*: https://www.codecentric.de/en/knowledge-hub/blog/modularization-the-easy-way-spring-modulith-with-kotlin-and-hexagonal-architecture

**Concurrency (coroutines vs virtual threads):**
- JDriven — *From Java to Kotlin, Part X: Virtual Threads and Coroutines* (2026): https://jdriven.com/blog/2026/02/Java-To-Kotlin-Series-10-Threads
- Jorge Gonzalez — *Comparing Kotlin Coroutines and Java Virtual Threads with Spring Boot Examples*: https://medium.com/@jorgegfx/harnessing-modern-concurrency-comparing-kotlin-coroutines-and-java-virtual-threads-with-spring-cf79e234b892
- Kotlin Discussions — *Spring, Coroutines, Virtual Threads*: https://discuss.kotlinlang.org/t/spring-coroutines-virtual-threads/29949
- Benchmarks: https://github.com/gaplo917/coroutine-reactor-virtualthread-microbenchmark

**HTTP clients:**
- MEsfandiari — *The End of RestTemplate: Spring RestClient vs. WebClient vs. Retrofit in 2026*: https://medium.com/@mesfandiari77/the-end-of-resttemplate-spring-restclient-vs-webclient-vs-retrofit-in-2026-0ca96c463544

---

Build it once without AI autocomplete, narrating every stop out loud like the interview already started. Enjoy the ride. 🚀
