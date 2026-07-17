# Curriculum refinement backlog

Working document for the lesson-by-lesson refinement process (started 2026-07-16, generated from the full curriculum review). **One lesson per session.** Any session that picks up this work reads this file first, then follows the session ritual below. Delete this file when every box is checked.

## Session ritual

1. Read [lesson-template.md](lesson-template.md) — the mandatory format and authoring rules.
2. Read this file's section for the target lesson, plus the cross-cutting gaps below.
3. Read the lesson file itself and the SDD anchors it should link ([bookstore-api-sdd.md](../../bookstore-api-sdd.md)) — link ADRs, never restate them.
4. Rewrite the lesson to the template, applying the lesson's findings.
5. Verify ecosystem claims (versions, starter names, framework behavior) against current sources before asserting them — especially anything marked *(verify)*.
6. Mark the lesson `[x]` here, note the date, and summarize what changed.

## Authoring style (user-approved, do not drift)

- Code blocks **teach concepts**; compilable completeness is not required. Elide the unimportant with `// ...`; never elide the mechanism being taught. Comments/context must make the point of every block obvious.
- Implementation path = **ordered task list**, each task with a concrete *why* plus descriptive sub-steps that guide the hands ("create class X in package Y", "annotate the field with `@Z`", "register bean W in the bootstrap `@Configuration`"). Not a full code walkthrough.
- Verification is **cheap and proportionate**: one curl call, one SQL check, or one test run.
- Stack section names **central starters/libraries by Spring project name** (no full Maven coordinates, no exhaustive lists) and what each provides for the lesson. Domain lessons state "No new dependencies."
- Kotlin-first form first, traditional imperative twin second, as a comparison pair.
- 3–5 check-yourself questions per lesson; header block with effort/prerequisites/SDD anchors.

## Verified stack facts (as of 2026-07)

- Spring Boot **4.1.0** (released 2026-06-10) on Framework **7.0.8**; Kotlin **2.3**. Boot 4.1 added gRPC auto-config, HTTP-client SSRF mitigation (`InetAddressFilter`), improved OpenTelemetry support.
- **Boot 4 modularized starters**: `spring-boot-starter-web` → `spring-boot-starter-webmvc`; per-feature test starters (e.g. `spring-boot-starter-webmvc-test`); raw libraries that used to auto-configure (flyway-core, drivers) now need their own starters; classic starters exist as a transition aid. Boot 3-era tutorials must not be copied.
- Framework 7: native **API versioning** (`ApiVersionConfigurer`, `@GetMapping(version = "…")`), **JSpecify** null-safety (real Kotlin nullability from Spring APIs), resilience (`@Retryable`, `@ConcurrencyLimit`, `@EnableResilientMethods`; `@Retryable` adapts to reactive return types), **RestTestClient**, HTTP interface clients (`@GetExchange` + `@ImportHttpServices`).
- *(verify per session)*: spring-kafka suspend `@KafkaListener` semantics/ack modes; exact Modulith artifact names.

## Cross-cutting gaps (check against every lesson)

- **G1** — every lesson needs the Stack and dependencies section (template §5).
- **G2** — concept-code density: each lesson shows its central mechanism (per the style above).
- **G3** — SDD Part III (K0–M11) duplicates lessons; fold into a mapping table + remove the misleading ✅ heading glyphs. Guided tour already documents the transition. (Separate session, after the core track.)
- **G4** — verification loop: tasks close with cheap checks (template §8).
- **G5** — industry holes to place somewhere: CI workflow (Core 12), OpenTelemetry/Micrometer Tracing (Core 12, reused by Kafka/Saga tracks), Boot structured logging (Core 12), OpenAPI/springdoc (Core 08), ktlint/detekt (Core 03 or a tooling note), security-seam page (one pager, location TBD), `bootBuildImage` packaging (Core 12), load-measurement exhibit (showroom).
- **G6** — effort estimates in headers; check-yourself questions per lesson.

## Status and per-lesson findings

### Meta

- [x] `reference/lesson-template.md` — rewritten 2026-07-16: 8 authoring rules, header block, 14 sections (stack, task-list implementation path with sub-steps, common mistakes, check yourself).
- [x] `bookstore-guided-tour.md` — 2026-07-16: sources-of-truth table, indicative effort column, template anatomy, updated how-to-work loop.
- [ ] `bookstore-tour/README.md` — add per-phase effort, make "every lesson must prove" a copyable checklist.
- [ ] `bookstore-api-sdd.md` (G3) — Part III → mapping table to lessons; remove ✅ glyphs; bootstrap line must name Boot 4 modular starters.

### Core track

- [ ] **00 business context** — ✅ nearly complete. Add: header block; small actors/flow view; explicit SDD section links; check-yourself questions. "No new dependencies." Light session (~1 h).
- [ ] **01 kotlin domain foundations** — 🟡. Add: the promised sealed decision type in code; 1–2 decisive tests (Isbn rejection, order-total math); `value class` boxing nuance; **Money arithmetic pitfalls** (scale, rounding mode, currency mismatch on plus); stack section teaching that *zero dependencies is the point* (kotlin-test/JUnit only).
- [ ] **02 coroutines and flow** — 🔴 most under-built load-bearing lesson (carries the whole K1/K2 runway in 55 lines). Add: `runTest` + virtual-time example; cooperative cancellation in code (`ensureActive`, rethrow `CancellationException`); dispatcher injection as constructor parameter; cold Flow with `take(1)` cancellation; `coroutineScope` vs `supervisorScope` failing-sibling diff. Consider splitting into two lessons (structured concurrency / context+Flow+testing) or tripling depth. Deps: kotlinx-coroutines-core + kotlinx-coroutines-test.
- [ ] **03 first annotated CRUD** — 🔴 dependencies lesson without dependencies. Add: full stack table (`spring-boot-starter-webmvc`, Data JDBC, Flyway + postgres driver, validation, docker-compose support, Testcontainers, per-feature test starters); thin-slice concept code (controller/service/repo/DTO) with `@field:` callout and `ResponseEntity.created(location)`; the **corrected compose.yaml shown** (lesson currently says "reconcile" without target state); curl verifications for all five endpoints; `@SpringBootTest` + `@ServiceConnection` smoke test.
- [ ] **04 DDD and domain model** — 🟡. Add: before/after code-diff (anemic row-shaped service vs behavior-owning aggregate — the lesson's core move); small context-map view (upstream/downstream, sets up Kafka track); one concrete domain-event type.
- [ ] **05 use cases, ports, services** — 🟡 closest to done. Add: `CreateBookCommand` definition + the Bean-Validation vs domain-validation boundary; MockK `coEvery`/`coVerify` + `runTest` snippet; the grouped-queries alternative it endorses. Deps: MockK, coroutines-test.
- [ ] **06 hexagonal refactor** — 🔴 ArchUnit is the lesson and isn't there. Add: complete `HexagonalRulesTest` (domain purity, ring access, stereotype location, naming); archunit-junit5 dep; refactor order (which packages move first, tests stay green); Kotlin `internal` as a cheap boundary tool.
- [ ] **07 data JDBC adapters** — 🟡 best code coverage already. Add: `OrderRow`/`OrderItemRow` + `@MappedCollection`; stable-string enum converter code; the actual `V1__create_books.sql`; `@DataJdbcTest` + Testcontainers snippet; **`jdbcDispatcher` bean definition** in bootstrap (`limitedParallelism`) — referenced everywhere, defined nowhere; JSpecify nullability payoff note. Deps block.
- [ ] **08 http contracts and errors** — 🟡. Add: `@RestControllerAdvice` code + sample ProblemDetail JSON body; **API versioning config** (`ApiVersionConfigurer` + versioned mapping + curl with header) — flagship Spring 7 demo; `@field:` trap in code; `@WebMvcTest` example (webmvc-test starter); springdoc/OpenAPI here (G5).
- [ ] **09 catalog queries** — 🟡. Add: `JdbcClient` projection query code (centerpiece); `PageQuery`/`PagedResult` definitions; sort-whitelist implementation (security-relevant); keyset-pagination cursor sketch; one `EXPLAIN` exercise.
- [ ] **10 orders, transactions, locking** — 🔴 the meaty lesson is missing its meat. Add: commit adapter + **separate proxied `@Transactional` delegate** pair in code (no suspension inside tx, visible); two-thread race test (`CountDownLatch`/barrier + asserts); `Order` state machine; resolve the dangling idempotency mention (teach Idempotency-Key here or explicitly defer to Saga track with link).
- [ ] **11 outbound http and resilience** — 🟡. Add: HTTP-interface trio (`@GetExchange` interface, `@ImportHttpServices`, adapter `awaitSingle()` inside `Semaphore.withPermit`); reactive `@Retryable` collaborator; `spring.http.serviceclient.*` timeout yaml; name the stub tool (WireMock or MockWebServer) + dep; **circuit-breaker paragraph** (Framework 7 has retry/limits, no CB — when Resilience4j enters).
- [ ] **12 observability, testing, release** — 🟡. Add: MDC `OncePerRequestFilter` + log pattern; one Micrometer counter (optimistic-lock conflicts); test proving MDC after suspension; Boot **structured logging**; **OpenTelemetry** section; minimal **CI workflow** (GitHub Actions: build + Testcontainers + ArchUnit); per-feature test starter note; `bootBuildImage` mention (G5).

### WebFlux fulfillment track

- [ ] **README** — 🟡. State it is a separate Gradle project; how to run both apps (ports, compose); stack block (`spring-boot-starter-webflux`, Data R2DBC, r2dbc-postgresql, Flyway-over-JDBC).
- [ ] **00 context and reactive decision** — ✅ nearly complete. Add concrete target numbers (subscribers, events/min) so later load tests have criteria.
- [ ] **01 domain, use cases, ports** — 🟡. Add: transition state machine in code (sealed states + `transitionTo` returning a sealed result).
- [ ] **02 r2dbc persistence** — 🟡. Add: row + `CoroutineCrudRepository` code; `TransactionalOperator.executeAndAwait` example; **dual-URL yaml** (Flyway JDBC + runtime R2DBC) — the #1 newcomer stumbling block.
- [ ] **03 webflux, flow, SSE** — 🔴 real design hole: **the fan-out mechanism is never chosen** (R2DBC doesn't push — in-process `SharedFlow` after commit vs Postgres LISTEN/NOTIFY vs polling). Needs a Decision section + concept code: controller `Flow` with `TEXT_EVENT_STREAM`, heartbeat merge, buffer/overflow policy, Last-Event-ID contract, WebTestClient streaming test sketch.
- [ ] **04 carrier webhooks** — 🟡 strongest security content. Add: HMAC signature-verification snippet; inbox table sketch; unknown-status mapping function.
- [ ] **05 reliability, testing, performance** — 🟡. Name the tools: Gatling or k6 for load, Toxiproxy (Testcontainers) for disconnect chaos.
- [ ] **06 functional routing** — ✅ complete for purpose; only template conformance (header, stack line, check-yourself).

### Kafka / EDA track

- [ ] **README + 00** — ✅ nearly complete. Add stack block at README (spring-kafka, Spring Modulith BOM, Kafka Testcontainers); staging diagram optional.
- [ ] **01 domain vs integration events** — 🟡. Add: concrete `OrderPlacedV1` (data class + JSON golden sample + boundary mapping fn); **schema-management paragraph** (plain JSON + contract tests now; when Avro/Schema Registry earns its place).
- [ ] **02 modulith and outbox** — 🔴 flagship not coded. Add: `@ApplicationModuleListener` example; `@Externalized` externalization; publication-registry config (completion mode, republish policy); explicit outbox-table variant sketch; **module declaration + `ApplicationModules.verify()`** (never taught anywhere); deps (spring-modulith-starter-jdbc, -events-kafka, -events-api) *(verify artifact names)*.
- [ ] **03 kafka publisher** — 🟡. Add: producer config yaml (acks=all, idempotence, compression); publisher adapter code; topic-creation strategy (`KafkaAdmin`/`NewTopic` vs ops-owned); Testcontainers-Kafka test sketch.
- [ ] **04 single idempotent consumer** — 🟡. Add: `@KafkaListener` + ack-mode decision in code; inbox DDL + unique constraint; `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` wiring.
- [ ] **05 groups and ordering** — ✅ nearly complete. Add: two-group config example; container concurrency property.
- [ ] **06 service extraction** — 🟡. Add: **data backfill/migration** step (historical orders → new service) — the step every real extraction hits; what shadow-mode concretely compares.

### Sagas track

- [ ] **README** — ✅ fine; template conformance only.
- [ ] **00 distributed checkout context** — ✅ nearly complete. Give the **draft-order refactor** (PlaceOrder split into draft + checkout) a home: short section or new mini-lesson.
- [ ] **01 semantics and compensations** — ✅ complete as concept lesson. Add Kotlin envelope types to make identifiers concrete.
- [ ] **choreography/01 event flow** — 🟡. Add: sequence diagram; topic/key/partition table per event; ordering's operation-tracking state sketch.
- [ ] **choreography/02 participants** — 🟡. Add: inbox+outbox one-local-transaction code; pin spring-kafka version + ack mode for suspend `@KafkaListener` *(verify support semantics before asserting)*.
- [ ] **choreography/03 failures and recovery** — 🟡. Add: reconciler sketch; timeline query shape.
- [ ] **orchestration/01 coordinator flow** — 🟡. Add a state diagram; otherwise close.
- [ ] **orchestration/02 coordinator implementation** — 🟡. Add: saga table DDL; one transition function in code; reply-correlation consumer sketch.
- [ ] **orchestration/03 durability and recovery** — 🟡. Add: scheduler idiom (`@Scheduled` poll + `FOR UPDATE SKIP LOCKED` — name the industry pattern); operator-endpoint sketch; name **Temporal/Camunda** + adoption criteria (here or in 90).
- [ ] **90 comparison** — ✅ complete. Optionally add the workflow-engine names/criteria if not placed in orchestration/03.

### Event sourcing / CQRS track

- [ ] **README + 00** — ✅ complete; template conformance only.
- [ ] **01 event store and rehydration** — 🟡. Add: event-table DDL (both unique constraints); sealed event hierarchy + `fold`/`apply` rehydration; expected-version conditional append; given/when/then test helper.
- [ ] **02 CQRS projections** — 🟡. Add: projection-position table + one idempotent upsert handler; rebuild-then-cutover numbered runbook.
- [ ] **03 trade-offs and evolution** — ✅ nearly complete. Name **crypto-shredding** for erasure-in-immutable-history; one upcaster sketch.

### Showroom and reference

- [ ] **showroom** — 🟡. Add a "how to wire a variant" page (`@Profile`/`@ConditionalOnProperty` adapter selection in code); measurement exhibit (Gatling/k6 harness) fits here (G5).
- [x] **annotation-and-role-guide** — ✅ complete as-is (2026-07-16 review).
- [x] **decision-guide** — ✅ complete as-is (2026-07-16 review).
- [ ] **further-reading** — 🟡. Add: Boot 4 migration guide + modularization blog, Spring Modulith reference, JSpecify, Boot structured-logging docs, OTel/Micrometer Tracing docs.

## Recommended order

Core 00 → 12 in sequence (🔴 lessons 02, 03, 06, 10 get the deepest passes) → WebFlux (settle 03's fan-out decision early) → Kafka (02 is the heavy one) → Sagas → Event sourcing → showroom/reference → SDD Part III fold (G3) → cross-cutting additions not yet placed.
