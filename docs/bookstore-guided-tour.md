# The Bookstore Build
## Kotlin-first Spring Boot 4 curriculum

This file is the front door to the curriculum. The original all-in-one tour has been split so that each lesson starts with a business problem, introduces only a small set of skills, leaves the code runnable, and explains when the technique should and should not be used.

The project grows through three deliberately different shapes:

1. A **Spring MVC modular monolith** for catalog and ordering.
2. A separate **WebFlux fulfillment API** where streaming and non-blocking IO have a business reason to exist.
3. An **advanced distributed-systems path** that evolves module events into Kafka, extracts services, compares choreographed and orchestrated Sagas, and finishes with selective event sourcing/CQRS.

Start with the [curriculum map](bookstore-tour/README.md). Do not begin with Kafka, WebFlux, or Sagas: those lessons assume the domain, ports, transaction, cancellation, and failure semantics established earlier.

## Sources of truth

Each document owns one job. When two documents seem to disagree, the owner wins — and the other document has a bug worth fixing.

| Document | Owns | Does not own |
|---|---|---|
| [SDD](bookstore-api-sdd.md) | Scope, ADRs, annotation policy, tech-stack contract, definition of done | Step-by-step implementation |
| [Curriculum map](bookstore-tour/README.md) | Learning order and phase gates | Architecture decisions |
| Lessons under `bookstore-tour/` | The how-to: per-lesson stack, steps, verification, tests, traps | Restated ADRs — lessons link to them |
| [Repository README](../README.md) | Current implementation status of the codebase | Curriculum content |

The SDD's Part III milestone list (K0–M11) is being folded into the lessons; until that migration completes, treat overlapping milestone text as a pointer to the corresponding lesson, not as a second curriculum.

## Required path

| Phase | Outcome | Indicative effort |
|---|---|---|
| [Core](bookstore-tour/core/00-business-context.md) | A professional annotated MVC API with Kotlin-first domain code, use cases, ports/adapters, JDBC, transactions, resilience, observability, and tests | 25–40 h |
| [WebFlux fulfillment](bookstore-tour/webflux-fulfillment/README.md) | A second API that owns live order fulfillment and exposes an end-to-end reactive path with R2DBC and `Flow`/SSE | 12–20 h |
| [Kafka and EDA](bookstore-tour/advanced/kafka-eda/README.md) | Reliable external events, consumers, groups, and the first bounded-context extraction | 14–22 h |
| [Sagas](bookstore-tour/advanced/sagas/README.md) | The same distributed checkout implemented with choreography and orchestration | 16–26 h |
| [Event sourcing and CQRS](bookstore-tour/advanced/event-sourcing-cqrs/README.md) | A selective fulfillment experiment, not a rewrite of the whole bookstore | 8–14 h |

Effort figures are planning aids at a focused pace, not deadlines; deep experiments legitimately double them. Per-lesson estimates live in each lesson's header block.

## Showroom and reference material

- [Technology showroom](bookstore-tour/showroom/README.md) compares persistence, web, client, and runtime choices without leaking technology names into public resource URLs (optional, ~4–8 h).
- [Lesson template](bookstore-tour/reference/lesson-template.md) defines the mandatory context-first lesson format: a header with effort and prerequisites, a stack table naming the central starters and what they provide, concept-teaching code, an ordered implementation task list with cheap verifications, common mistakes, and check-yourself questions.
- [Annotation and role guide](bookstore-tour/reference/annotation-and-role-guide.md) maps Spring stereotypes to hexagonal roles.
- [Decision guide](bookstore-tour/reference/decision-guide.md) summarizes when to use or avoid each approach.
- [Further reading](bookstore-tour/reference/further-reading.md) contains the primary research trail.

## The dependency rule

```text
HTTP or Kafka input adapter
        -> input port
        <- @Service application service (use-case interactor)
        -> output port
        <- @Repository/@Component output adapter
        -> database, broker, or remote service
```

The domain remains plain Kotlin. `@RestController`, `@Service`, `@Repository`, and `@Component` are welcome when they describe a real Spring runtime role; adapter implementation types never point inward.

## Naming convention

Use Spring-friendly service names such as `CreateBookService`, not `CreateBookInteractor` or `CreateBookServiceImpl`. The use-case lesson explicitly teaches that the service **is the Clean Architecture interactor**:

```kotlin
/**
 * Application service implementing the CreateBook use case.
 * In Clean Architecture terminology, this is the use-case interactor.
 */
@Service
class CreateBookService(...) : CreateBookUseCase
```

## How to work through a lesson

Read the problem and acceptance criteria first, then the stack table so every dependency you add has a stated purpose. Work the implementation task list in order, closing each task with its quick verification — a curl call, a SQL check, a test run — before moving on. Run the lesson's tests, inspect the package tree, and finish by answering the check-yourself questions aloud. A lesson is incomplete if the code runs but you cannot explain its blocking points, failure behavior, operational cost, or architectural trade-offs.
