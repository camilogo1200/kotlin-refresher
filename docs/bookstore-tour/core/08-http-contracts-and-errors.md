# Core 08 — HTTP contracts, validation, and errors

## Business context

Mobile, terminal, and partner clients evolve independently from internal models. They require stable requests, predictable validation, and one machine-readable error shape.

## Acceptance criteria

- Controllers expose request/response DTOs only.
- Create returns `201` with `Location`; delete returns `204`.
- Validation failures identify fields.
- Missing resources, conflicts, and unexpected failures use RFC 9457 Problem Details.
- One endpoint demonstrates Spring Framework 7 API versioning.

## HTTP adapter responsibilities

```text
HTTP request
  -> request DTO + Bean Validation
  -> command/query
  -> input port
  -> response DTO or application error
  -> HTTP response/ProblemDetail
```

Use explicit Kotlin use-site targets such as `@field:NotBlank` when the validation target matters. Keep controllers thin; status codes, headers, DTO mapping, and authentication context belong here, while business rules do not.

## Error mapping

One `@RestControllerAdvice` maps:

- `BookNotFound` to `404`.
- Duplicate ISBN, insufficient stock, and optimistic conflict to `409`.
- Request validation to `400` with field-error properties.
- Unexpected exceptions to a generic `500` without internal details.

Domain and application errors remain transport-neutral; the advice decides HTTP semantics.

## Versioning

Keep a coarse `/api/v1` path and demonstrate Spring 7 native version selection on a changed representation. Configure one strategy consistently; do not mix arbitrary header, media type, and query conventions.

## Tests and proof

Use `@WebMvcTest` with mocked input ports. Assert JSON contracts, status, headers, validation shape, error content type, version routing, and absence of internal fields.

## Do and don't

- **Do:** version contracts when client-visible semantics change.
- **Don't:** version merely because implementation code changed.
- **Don't:** leak stack traces, persisted rows, or domain internals.

Continue to [catalog queries](09-catalog-queries.md).

## Performance and production note

Bound request sizes and validation complexity before deserialization becomes a memory/CPU attack surface. Error handling should avoid blocking work and high-cardinality metrics. Stable DTOs reduce client and deployment coordination cost more than exposing internal types saves code.
