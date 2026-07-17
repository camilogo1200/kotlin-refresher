# Core 07 — Spring Data JDBC output adapters

## Business context

The application now owns repository contracts, but production needs durable aggregate persistence with explicit SQL lifecycle and optimistic locking. Kotlin model design must not be distorted by ORM proxy requirements.

## Decision

Use Spring Data JDBC for the core monolith. Keep immutable storage rows and internal Spring Data repositories inside the adapter.

```kotlin
@Table("books")
data class BookRow(
    @Id val id: Long? = null,
    val isbn: String,
    val title: String,
    val author: String,
    val price: BigDecimal,
    val stock: Int,
    @Version val version: Long? = null,
)

internal interface SpringDataBookRepository : CrudRepository<BookRow, Long>

@Repository
class JdbcBookRepository(
    private val repository: SpringDataBookRepository,
    @Qualifier("jdbcDispatcher") private val jdbcDispatcher: CoroutineDispatcher,
) : BookRepository {
    override suspend fun findByIsbn(isbn: Isbn): Book? =
        withContext(jdbcDispatcher) { repository.findByIsbn(isbn.value)?.toDomain() }
}
```

The generated Spring Data repository needs no decorative `@Repository`; the architectural adapter gets the stereotype.

## Aggregate mapping

An `OrderRow` owns its `OrderItemRow` collection through `@MappedCollection`. Cross-aggregate references are identifiers. Data JDBC loads an aggregate explicitly and has no lazy-loading session.

Flyway owns all DDL. Money maps to a fixed-precision numeric type. Persist enum values with an explicit stable-string converter, never an ordinal.

## Blocking bridge

JDBC still blocks a thread and occupies a connection. `withContext(jdbcDispatcher)` moves that blocking to an adapter-owned policy; it does not make JDBC reactive. Align dispatcher parallelism with measured pool/downstream capacity rather than copying a magic number.

## Tests and proof

Use `@DataJdbcTest`, Flyway migrations, and Testcontainers PostgreSQL. Prove constraints, numeric mapping, aggregate children, generated ids/version handling, and optimistic-lock behavior against the real dialect.

## When to choose something else

- Use `JdbcClient` when explicit SQL and projections matter more than aggregate repository convenience.
- Use JPA for an ORM-heavy domain that benefits from its unit of work and mappings, accepting proxy/fetch complexity.
- Use R2DBC only when the complete workload benefits from non-blocking database IO.

Continue to [HTTP contracts and errors](08-http-contracts-and-errors.md).

