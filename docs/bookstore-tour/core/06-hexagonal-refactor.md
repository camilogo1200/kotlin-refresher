# Core 06 — Feature-first hexagonal refactor

## Business context

Catalog and ordering now change for different reasons. Global `controller`, `service`, and `repository` folders make features hard to navigate, while simple layering does not prevent adapter types from leaking inward.

## Decision

Organize by business feature first and hexagonal role inside the feature:

```text
com.example.kotlinrefresher
├── bootstrap/
├── catalog/
│   ├── domain/
│   ├── application/
│   │   ├── port/input/
│   │   ├── port/output/
│   │   └── usecase/
│   └── adapter/
│       ├── input/http/
│       └── output/persistence/jdbc/
├── ordering/
│   ├── domain/
│   ├── application/
│   └── adapter/
└── shared/
```

Use `input` and `output`; `in` is a Kotlin keyword. Keep one Gradle module until a real build/deployment boundary earns another.

## Dependency rules

| Package | May depend on | Must not depend on |
|---|---|---|
| Domain | Kotlin/JDK basics | Spring, Jakarta, application, adapters |
| Application | Domain, ports, coroutines, Spring stereotypes | HTTP, Spring Data, Kafka, output implementations |
| Input adapter | Input ports, mapping, web/messaging framework | Output adapter implementations |
| Output adapter | Output ports, domain, technology | Input adapters |
| Bootstrap | All rings for composition | Business decisions |

Enforce these rules with ArchUnit. Folder names alone do not create an architecture.

## Spring stereotype policy

- `@RestController` and `@RestControllerAdvice`: HTTP input adapters.
- `@Service`: application use-case implementations.
- `@Repository`: persistence adapters.
- `@Component`: other adapters/infrastructure when no narrower stereotype fits.
- `@Configuration`, `@Bean`, `@ConfigurationProperties`: bootstrap.
- No framework annotations in domain.

## Cost accounting

Hexagonal structure adds files and mapping. It pays when business logic, multiple entry points, alternative adapters, or long maintenance justify those costs. For one trivial CRUD table, the Core 03 structure can be sufficient.

## Exit criteria

Packages follow the feature-first tree, dependency rules fail when violated, and the application still runs. Continue to [Data JDBC adapters](07-data-jdbc-adapters.md).

## Performance and production note

Mapping between domain and adapters has a measurable but normally minor allocation cost. Profile before collapsing boundaries. More important operational wins are independent adapter tests, explicit blocking points, and smaller change blast radius.
