# Core 03 — First annotated CRUD slice

## Business context

Catalog staff need a usable API quickly. Before introducing full ports-and-adapters structure, build one thin vertical slice to refresh ordinary Spring Boot annotations and establish a working baseline.

## Request

Implement create, get, update, delete, and list operations for books using an intentionally small structure:

```text
BookController @RestController
    -> BookService @Service
        -> SpringDataBookRepository
            -> PostgreSQL
```

This is scaffolding, not the final architecture.

Before adding the route, reconcile the walking skeleton: make Docker Compose configuration valid, remove or configure the out-of-scope Redis service, boot PostgreSQL, apply the first Flyway migration, and bind one immutable `@ConfigurationProperties` class. Prove `docker compose --env-file .env.example config --quiet`, application startup, migration, and health before calling CRUD runnable.

## Acceptance criteria

- `POST /api/v1/books` returns `201` and a `Location` header.
- Get/update/delete distinguish missing books.
- Requests use Bean Validation; responses never expose a Spring Data row directly.
- Flyway owns the schema.
- Typed configuration and the local Compose profile boot deterministically.
- A smoke integration test proves the slice works against PostgreSQL.

## Skills

- `@RestController`, `@Service`, repository discovery, constructor injection.
- Spring Data JDBC `@Table`, `@Id`, immutable rows, and Flyway.
- Request/response data classes and Kotlin annotation use-site targets such as `@field:NotBlank`.

## Shortcuts recorded as debt

The first `BookService` may depend directly on the Spring Data repository so the initial request can ship. Record the consequences:

- Application behavior knows a persistence API.
- Repository return types can leak into orchestration.
- Tests are tempted to mock infrastructure instead of business-facing contracts.
- Adding a second adapter will be awkward.

The next lessons introduce abstractions only when this pressure is visible.

## Do and don't

- **Do:** use the most specific stereotype and constructor injection.
- **Do:** keep DTO mapping explicit even in the quick slice.
- **Don't:** create `BookServiceImpl` or a matching interface solely for convention.
- **Don't:** call this clean architecture yet.

## Exit criteria

CRUD is runnable, validated, and migrated. You can identify the shortcuts that motivate [DDD](04-ddd-and-domain-model.md) and ports.

## Performance and production note

This slice is sufficient for modest CRUD workloads when queries are indexed and connection/page limits are bounded. Avoid adding architectural layers merely to improve throughput—they do not. Add them when change, ownership, and testing pressure appears.
