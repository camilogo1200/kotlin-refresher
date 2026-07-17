# Lesson template

Use this structure for every lesson. Context comes before technology; code exists to teach concepts, and an ordered task list drives the implementation.

## Authoring rules

1. **Context first.** No technology appears before the business pressure that justifies it.
2. **Code teaches the point.** A code block carries enough context — naming, comments, a sentence before or after — that the developer knows what the block demonstrates and why it matters. Elide what is not important with `// ...`; never elide the mechanism the lesson exists to teach. Compilable completeness is not required.
3. **Kotlin-first pairs.** When a traditional imperative Spring alternative exists, show the coroutine/Kotlin-first form first, then the traditional twin as a comparison pair. The diff is the lesson.
4. **Verify cheaply.** Every implementation task ends with the cheapest check that proves it: a `curl` call, a SQL query against the database, or a single test run. Verification is proportionate to the lesson; do not build ceremony around it.
5. **Link, don't restate.** Architecture decisions live in the [SDD](../../bookstore-api-sdd.md) ADRs; lessons link to them. Restated decisions drift.
6. **Three or four new skills maximum.** Link prerequisites instead of repeating them.
7. **Traps are content.** Every lesson records the mistakes it tempts, especially the ones that fail silently.
8. **Concept-only lessons** (business context, track intros, comparisons) may compress sections 7–11, but must still fill section 5 (even if it is "No new dependencies"), section 12, and section 14.

## Header block

Open every lesson with:

| | |
|---|---|
| **Estimated effort** | e.g. 2–4 focused hours |
| **Prerequisites** | links to the required lessons |
| **SDD anchors** | the ADRs/sections this lesson implements |

Estimates are honest planning aids at a focused pace; taking longer is normal, not falling behind.

## 1. Business context

Who is asking, what currently fails, why it matters, and what cost or risk exists?

## 2. Request and acceptance criteria

State observable outcomes. Include failure and concurrency behavior where relevant.

## 3. Current state and gap

Describe what already works and the specific pressure that justifies a new concept.

## 4. Decision

Name the selected approach, alternatives considered, and why this lesson selects it. Link the ADR instead of repeating its argument.

## 5. Stack and dependencies

Name the central Spring starters and libraries the lesson uses — Spring project names are enough; exact coordinates and versions belong to the build, not the lesson. Describe what each one does for this lesson. List only what is central; transitive and incidental dependencies stay out.

| Starter / library | What it provides in this lesson |
|---|---|
| `spring-boot-starter-webmvc` | Annotated MVC endpoints (Boot 4 modular starter; replaces `spring-boot-starter-web`) |
| Spring Data JDBC + Flyway | Aggregate persistence and the migration that owns the schema |
| Testcontainers (PostgreSQL) | Integration tests run against the real database engine |

Write **"No new dependencies."** explicitly when true — in domain lessons that absence is the point.

## 6. Concepts before code

Introduce the new skills (three or four at most) with the smallest example that shows each one honestly.

## 7. Interaction and dependency flow

Show controller/listener, input port, `@Service`, output port, and adapter relationships for the slice being built.

## 8. Implementation path

An ordered task list, not a full code walkthrough — a lesson cannot and should not contain every file. Each task states the concrete **why**, then a short set of descriptive steps that guide the hands — create this class in this package, add this annotation to that class or field, register this bean in the `@Configuration` class — so the developer writes the code while the lesson steers. Code appears only where it teaches the concept. Close each task (or logical group of tasks) with a cheap verification.

````markdown
1. **Create the Flyway migration for `books`** — the schema must exist before any
   mapping, and Flyway owning DDL is this lesson's persistence rule.
   - Add `V1__create_books.sql` under `src/main/resources/db/migration`.
   - Define isbn (unique), title, author, price `NUMERIC(10,2)`, stock, and a
     `version` column for optimistic locking.
2. **Add `BookRow` and the internal Spring Data repository** — the immutable row
   keeps persistence shape out of the domain.
   - Create data class `BookRow` in `adapter/output/persistence/jdbc`; annotate
     the class with `@Table("books")` and the id property with `@Id`.
   - Add `@Version val version: Long? = null` so the lock column is mapped from day one.
   - Create `internal interface SpringDataBookRepository : CrudRepository<BookRow, Long>` —
     `internal` so it cannot leak past the adapter.
3. **Implement `JdbcBookRepository` against the output port** — the adapter is
   where the technology meets the contract.
   - Create `JdbcBookRepository` implementing `BookRepository`; annotate the class
     with `@Repository`.
   - If it does not exist yet, register the `jdbcDispatcher` `@Bean` in the
     bootstrap `@Configuration`; inject it and wrap every blocking call:

   ```kotlin
   @Repository
   class JdbcBookRepository(/* ... */) : BookRepository {
       override suspend fun findByIsbn(isbn: Isbn): Book? =
           withContext(jdbcDispatcher) { repository.findByIsbn(isbn.value)?.toDomain() }
   }
   ```

**Verify** — the slice works end to end:

```bash
curl -i localhost:8080/api/v1/books/42        # 200 with JSON, 404 for unknown id
```

```sql
select isbn, stock, version from books;       -- the row and its version column exist
```
````

## 9. Tests and proof

List the cheapest tests that demonstrate domain behavior, adapters, integration, and architecture — and show the decisive one: the given/when/then and the assertion that proves the behavior.

## 10. Common mistakes and pro tips

Name each trap, show the broken shape, show the fix. Prefer traps that fail silently: `@field:` annotation targets, proxy self-invocation, a `Flow` collected under `application/json`. Pro tips are the judgment a senior reviewer would add to the pull request.

## 11. Performance and operations

Separate reasoned characteristics from measurements. Identify the limiting resource: CPU, threads, connections, memory, broker partitions, or downstream capacity.

## 12. Do, don't, and industry notes

Explain when to use the approach, when not to use it, common failure modes, and production practices.

## 13. Resulting structure and exit criteria

Show changed packages/files and a completion checklist. End with the business capability now available and the pressure that motivates the next lesson.

## 14. Check yourself

Three to five questions the learner answers out loud. Include at least one about failure behavior and one about when **not** to use the technique. These feed the SDD interview arsenal.
