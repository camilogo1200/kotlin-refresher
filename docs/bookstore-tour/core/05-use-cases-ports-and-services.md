# Core 05 — Use cases, ports, and application services

## Business context

The controller currently calls persistence-shaped APIs. A second entry point or repository implementation would duplicate orchestration or pull technology further inward. The application needs explicit capabilities and dependencies.

## Interface-to-implementation map

```text
POST /books
  BookController @RestController
      -> CreateBookUseCase                 input port
          <- CreateBookService @Service   use-case interactor
              -> BookRepository           output port
                  <- JdbcBookRepository @Repository
```

## Naming convention

Use `CreateBookService`, not `CreateBookInteractor` and not `CreateBookServiceImpl`:

```kotlin
interface CreateBookUseCase {
    suspend fun execute(command: CreateBookCommand): Book
}

/**
 * Application service implementing the CreateBook use case.
 * In Clean Architecture terminology, this is the use-case interactor.
 */
@Service
class CreateBookService(
    private val books: BookRepository,
) : CreateBookUseCase {
    override suspend fun execute(command: CreateBookCommand): Book = TODO()
}
```

The Spring name communicates the runtime role; the lesson teaches the architectural name.

## Port policy

- Input ports make capabilities greppable and let HTTP/Kafka adapters share behavior. They are useful in this teaching project, although small production services sometimes call concrete application services directly.
- Output ports are mandatory at technology boundaries and are owned inward.
- Name output ports in business language: `BookRepository`, `PriceProvider`, `NotificationPublisher`.
- Name implementations with technology: `JdbcBookRepository`, `WebClientPriceProvider`, `KafkaNotificationPublisher`.
- Spring Data repositories remain internal infrastructure behind the adapter.

## Tests and proof

Instantiate the `@Service` directly with fakes or MockK ports. Prove duplicate ISBN rejection, mapping from command to domain, and that persistence is not called after validation failure.

## Do and don't

- **Do:** create one command use case per meaningful behavior; group trivial queries pragmatically.
- **Don't:** expose `Pageable`, JDBC rows, `Mono`, Kafka records, or HTTP responses through application ports.
- **Don't:** add interfaces between classes in the same layer without a boundary or second implementation.

## Exit criteria

Every controller dependency, use-case implementation, output port, and adapter pairing is visible. Continue to the [hexagonal refactor](06-hexagonal-refactor.md).

## Performance and production note

A Kotlin interface call is not the meaningful performance cost here; database, serialization, and network work dominate. The production risk is abstraction noise: keep ports role-shaped and delete speculative interfaces that no longer clarify a boundary.
