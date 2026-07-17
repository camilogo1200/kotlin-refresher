# Technology showroom

The showroom compares approaches after the core API is understood. It is not a collection of technology-named public endpoints.

## Business rule

Public resources describe the bookstore:

```text
/books
/orders
/fulfillments
```

Avoid `/books-jdbc`, `/books-r2dbc`, or `/books-webflux`. Persistence and runtime are adapter choices, not client-facing business concepts.

## Exhibits

- [Persistence variants](persistence-variants.md): in-memory pure Kotlin, `JdbcClient`, Spring Data JDBC, JPA, and R2DBC.
- [Runtime and web variants](runtime-and-web-variants.md): annotated MVC, virtual threads, WebFlux annotations, functional routing, and Kotlin coroutines.

Run only one adapter variant for a port unless qualifiers/profiles make the selection explicit. Keep comparisons in dedicated profiles, source sets, branches, or companion applications so bean ambiguity and classpath auto-configuration do not become the lesson.

## Comparison method

For each exhibit, keep the business acceptance criteria and input port stable. Compare:

- Code shape and framework types at boundaries.
- Blocking points and scarce resources.
- Transaction model.
- Failure/cancellation behavior.
- Operational and testing cost.
- Migration effort and team familiarity.

Do not claim one winner for every workload.

