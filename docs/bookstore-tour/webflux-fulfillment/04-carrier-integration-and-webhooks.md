# Fulfillment 04 — Carrier integration and webhooks

## Business context

Some carriers push updates through webhooks; others require status lookup. Payloads, authentication, retry behavior, and status vocabularies vary by carrier.

## Inbound webhook adapter

`CarrierWebhookController @RestController` verifies the carrier signature, rejects stale/replayed requests, maps the carrier payload into `RecordTrackingEventCommand`, and calls the input port. Persist an external event id/inbox record so duplicate delivery is harmless.

## Outbound client adapter

`WebClientCarrierTrackingClient @Component` implements `CarrierTrackingClient`. Keep carrier DTOs, HTTP headers, Reactor types, and authentication in the adapter. Map external statuses into the bounded-context vocabulary explicitly; unknown values are observable failures, not silent defaults.

Use structured `async` fan-out only when querying multiple independent carriers or resources. A single call does not need `async`.

## Security and operations

- Authenticate webhooks with the carrier-supported signature/certificate scheme.
- Store secrets outside source control and rotate them.
- Bound body size and reject unexpected content types.
- Separate retryable transport failures from invalid signed business events.
- Do not log signatures, tokens, addresses, or full personal payloads.

## Tests

Prove valid/invalid signatures, replay rejection, duplicate idempotency, timeout, retry classification, unknown status mapping, and cancellation.

## Use, avoid, and performance notes

Prefer signed push webhooks when carriers support reliable delivery; poll only when required and rate-limit it. Avoid unbounded parallel carrier lookups. Monitor signature failures, duplicate rate, provider latency, webhook age, and reconciliation backlog.
