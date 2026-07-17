# Core 01 — Kotlin domain foundations

## Business context

Bookstore data arrives as strings and numbers, but the business does not accept every string or number. An ISBN has a shape, price must be positive, stock cannot become negative, and money cannot use binary floating-point arithmetic.

## Request and acceptance criteria

Create a plain Kotlin model that can reject invalid states before Spring, JSON, or SQL exists.

- ISBN validation is centralized.
- Money uses `BigDecimal` and explicit currency.
- Book stock changes only through behavior that preserves the invariant.
- Order totals are calculated from line snapshots, never accepted from clients.
- Exhaustive business outcomes use sealed types where they improve callers.

## Skills

- `@JvmInline value class`, `data class`, regular class, and `sealed interface`.
- Null safety, exhaustive `when`, expressions, immutable collections, and extension functions.
- Identity versus value semantics.

## Model direction

```kotlin
@JvmInline
value class Isbn private constructor(val value: String) {
    companion object {
        fun parse(raw: String): Isbn = Isbn(normalizeAndValidate(raw))
    }
}

data class Money(val amount: BigDecimal, val currency: Currency)

class Book(
    val id: BookId?,
    val isbn: Isbn,
    val title: String,
    val author: String,
    val price: Money,
    stock: Int,
) {
    var stock: Int = stock
        private set

    fun decreaseStock(quantity: Int) { /* enforce rules */ }
}
```

Use a regular class for an identity-bearing aggregate with controlled mutation. Use data classes for immutable values, commands, DTOs, and persisted snapshots. Do not make every type a data class by reflex.

## Tests and proof

Use plain JUnit/Kotlin tests, with no Spring context or mocks:

- Invalid ISBN is rejected.
- Zero or negative price is rejected.
- Stock cannot become negative.
- Empty order and non-positive quantity are rejected.
- Order total equals the sum of quantity times captured unit price.

## Do and don't

- **Do:** prefer creation functions when constructors cannot clearly express validation failure.
- **Do:** keep domain time behind `Clock` where current time affects rules.
- **Don't:** add Spring, Jakarta, Jackson, JDBC, or coroutine imports to the domain.
- **Don't:** create `Utils` classes for behavior that belongs on a type.

## Exit criteria

The model compiles and its invariants are proven without Spring. Continue to the [coroutine runway](02-coroutines-and-flow.md).

## Performance and production note

Value objects and validation usually cost far less than recovering corrupted state. Measure allocation-sensitive hot paths before replacing clear domain types with primitive strings and numbers; do not trade correctness for speculative micro-optimization.
