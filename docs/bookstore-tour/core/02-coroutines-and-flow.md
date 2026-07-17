# Core 02 — Coroutines, structured concurrency, and Flow

## Business context

Later, an enriched catalog response must retrieve an independent price quote and rating. The application also needs streaming APIs. Learning these through controllers would hide task ownership, cancellation, and result semantics, so this lesson uses plain Kotlin first.

## Acceptance criteria

- Independent operations run concurrently inside a structured scope.
- One child failure cancels its dependent siblings.
- Blocking work is visibly isolated.
- A cold `Flow` stops when its collector cancels.
- Tests use virtual time where applicable.

## `Job`, `Deferred<T>`, `join`, and `await`

```kotlin
suspend fun enrich(
    price: suspend () -> PriceQuote,
    rating: suspend () -> Rating,
): EnrichedBook = coroutineScope {
    val priceTask: Deferred<PriceQuote> = async { price() }
    val ratingTask: Deferred<Rating> = async { rating() }
    EnrichedBook(priceTask.await(), ratingTask.await())
}
```

- `launch` returns `Job` for work producing `Unit`.
- `async` returns `Deferred<T>` when concurrent work produces a value.
- `Deferred<T>.join()` waits for completion and returns `Unit`.
- `Deferred<T>.await()` waits, returns `T`, and surfaces failure.
- `awaitAll()` is the fail-fast collection operation.
- `coroutineScope` owns its children; never use `GlobalScope` here.

## Context, supervision, and Flow

- `withContext(blockingDispatcher)` relocates blocking; it does not make the call non-blocking.
- Use `Dispatchers.Default` for CPU work, not blocking IO.
- Preserve `CancellationException`; cancellation is normal control flow.
- Use `supervisorScope` only when sibling failure is intentionally independent.
- Use `suspend fun one(): T?` for zero-or-one asynchronous result and `fun many(): Flow<T>` for a lazy stream.

## Tests and proof

Use `runTest` to prove successful fan-out, sibling cancellation, timeout cleanup, supervision differences, cold Flow collection, and `take(1)` cancellation. `runBlocking` is a bridge at a blocking boundary, not something to place inside a Spring controller or service.

## Performance and industry notes

Coroutines reduce the cost and complexity of waiting only when the libraries underneath cooperate or blocking is isolated. They do not increase database connection capacity. Structured lifetime and cancellation are their main advantage over unrelated futures.

Use them for concurrent IO composition, cancellation, and streams. Avoid `async` for sequential work, unbounded fan-out, and `Flow` as a replacement for every `List`.

## Exit criteria

You can explain task ownership, `join` versus `await`, cancellation, dispatcher responsibility, and when Flow is appropriate. Continue to the [first annotated CRUD slice](03-first-annotated-crud.md).
