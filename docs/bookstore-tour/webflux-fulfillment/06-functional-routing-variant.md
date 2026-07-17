# Fulfillment 06 — Functional routing variant

## Business context

The annotated API is complete. The team wants to evaluate WebFlux functional routing without changing the domain or use cases.

## Variant

Use `coRouter` and suspending handler functions for the same contracts. Keep this in an isolated profile/source variant so annotated and functional routes do not collide.

```kotlin
@Configuration
class FulfillmentRoutes {
    @Bean
    fun routes(handler: FulfillmentHandler) = coRouter {
        "/api/v1/fulfillments".nest {
            GET("/{id}", handler::get)
            GET("/{id}/stream", handler::stream)
        }
    }
}
```

Compare discoverability, validation/error handling, filter composition, test style, and team familiarity. Functional routing does not make blocking work reactive and does not replace application ports.

## Recommendation

Keep annotated controllers as the course default because the curriculum intentionally teaches Spring stereotypes. Use functional routing where explicit route composition or Kotlin DSL style creates clear value.

## Performance and production note

Annotated versus functional routing is primarily an organization/test-style decision; neither fixes a blocking dependency. Choose one convention per API surface, benchmark only if routing overhead is proven relevant, and optimize downstream IO first.
