# Showroom — Runtime and web variants

## Problem

Teams often compare coroutines, virtual threads, MVC, and WebFlux as if they were interchangeable products. They turn different dials.

| Dial | Options | Question answered |
|---|---|---|
| Code shape | `suspend`/`Flow` or imperative methods | How is asynchronous composition expressed? |
| Web runtime | Spring MVC or WebFlux | Servlet or reactive request processing? |
| Thread runtime | Platform or virtual threads | What is the cost of blocking? |
| Driver | JDBC or R2DBC | Does database IO block a thread? |

## Recommended exhibits

- Annotated MVC with suspending controllers and adapter-owned JDBC bridge: core main line.
- Imperative MVC on virtual threads: comparison for blocking libraries and migration-friendly code.
- WebFlux annotated controllers with coroutines: fulfillment main line.
- WebFlux functional `coRouter`: optional routing-style variant after annotations.

## Do and don't

- **Do:** use MVC with `WebClient` when only outbound calls benefit from non-blocking IO.
- **Do:** use WebFlux when the entire workload has high concurrency, streaming, and non-blocking dependencies.
- **Don't:** call JDBC directly on an event-loop thread.
- **Don't:** expose `Mono`/`Flux` through a Kotlin application core merely because the adapter uses Reactor.
- **Don't:** benchmark without naming connection limits and downstream capacity.

