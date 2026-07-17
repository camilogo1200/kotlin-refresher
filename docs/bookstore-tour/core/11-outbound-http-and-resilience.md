# Core 11 — Outbound HTTP, structured fan-out, and resilience

## Business context

Order placement needs a price quote, and an enriched catalog response needs independent price and rating calls. Slow or failing dependencies must not exhaust the bookstore or leave orphan work running.

## Architecture

```text
EnrichBookService @Service
  -> PriceProvider                 output port
      <- WebClientPriceAdapter @Component
  -> RatingProvider                output port
      <- WebClientRatingAdapter @Component
```

Technical HTTP interfaces may return Reactor types inside the adapter. The application ports remain `suspend` and expose domain-facing results.

## Structured fan-out

Use `coroutineScope { async { ... } }` only for independent calls whose combined result is needed. If one required call fails, sibling work is cancelled. Use `supervisorScope` only if partial results are a deliberate product decision.

## Resilience policy

- Apply connect/read/overall timeouts at the adapter edge.
- Retry only transient, idempotent operations with capped attempts, backoff, and jitter.
- Limit concurrency around scarce downstream capacity with a coroutine `Semaphore` for suspended calls.
- Keep retrying proxy methods on a collaborator so calls cross the Spring proxy.
- Preserve cancellation and classify business rejection separately from technical failure.

The virtual-thread showroom uses `RestClient` and synchronous method-invocation limits. Do not copy synchronous `@ConcurrencyLimit` semantics onto a suspending method without proving the permit covers the asynchronous operation.

## Tests and proof

Use a stub HTTP server to prove request mapping, timeouts, retry eligibility, exhaustion, bulkhead release after cancellation, sibling cancellation, and correlation propagation after a real suspension point.

## Do and don't

- **Do:** keep transport DTOs and `Mono` inside the adapter.
- **Don't:** retry the entire order transaction or non-idempotent remote commands casually.
- **Don't:** create a new WebFlux server merely because the client uses `WebClient`.

Continue to [observability and release readiness](12-observability-testing-and-release.md).

