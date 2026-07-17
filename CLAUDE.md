# CLAUDE.md

Kotlin-first Spring Boot 4 learning repository. The curriculum in `docs/` is the product right now; the application code is intentionally not implemented yet (the docs are milestones, the repo README states current status).

## Active work: curriculum refinement

A lesson-by-lesson refinement of `docs/bookstore-tour/**` is in progress — **one lesson per session**.

Before creating or editing any file under `docs/`, read [docs/bookstore-tour/reference/refinement-backlog.md](docs/bookstore-tour/reference/refinement-backlog.md) and follow its session ritual, authoring style, and per-lesson findings exactly. The backlog is also the progress tracker: mark a lesson `[x]` there when its refinement is done.

## Sources of truth

- [docs/bookstore-api-sdd.md](docs/bookstore-api-sdd.md) — scope, ADRs, annotation policy, definition of done. Lessons link to ADRs, never restate them.
- [docs/bookstore-tour/reference/lesson-template.md](docs/bookstore-tour/reference/lesson-template.md) — the mandatory lesson format and authoring rules.
- [docs/bookstore-guided-tour.md](docs/bookstore-guided-tour.md) — curriculum front door and document-ownership map.

## Conventions that apply everywhere

- Lessons teach concepts with annotated code fragments (`// ...` elision is fine); implementation is driven by ordered task lists with descriptive sub-steps, closed by cheap verification (one curl, SQL check, or test run).
- Kotlin-first form first, traditional imperative Spring twin second, as a comparison pair.
- Stack sections name central Spring starters by project name (Boot 4 modular starters — `spring-boot-starter-webmvc`, not the Boot 3-era `spring-boot-starter-web`); no full Maven coordinates.
- Verify framework/version claims against current sources before asserting them; the backlog lists the verified facts and the ones still marked *(verify)*.
