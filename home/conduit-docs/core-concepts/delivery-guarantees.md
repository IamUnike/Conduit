---
icon: badge-check
---

# Delivery Guarantees

Conduit guarantees at-least-once delivery. If an event matches a subscribed endpoint, Conduit will attempt delivery until either the endpoint acknowledges success or the retry policy is exhausted. Conduit does not guarantee exactly-once delivery.

This distinction matters in practice. At-least-once means your receiving endpoint may, in rare cases, receive the same event delivery more than once — for example, if your endpoint processed a request successfully but a network failure prevented Conduit from receiving the 200 response, Conduit will retry, and your endpoint will receive a duplicate.

This is expected behavior, not a bug, and it's why every event includes a stable event\_id in its payload. Your endpoint should treat delivery as idempotent: use the event\_id to detect and safely ignore duplicate deliveries, typically by recording processed event IDs and checking against that record before acting on a new one.

At-least-once delivery is the standard model for webhook systems generally, including Stripe's and GitHub's. Building idempotent handlers is a one-time integration cost that eliminates an entire category of duplicate-processing bugs — see [**Guides → Handling idempotency**](../guides/integration-guides.md#handling-idempotency-on-the-receiving-end) on the receiving end for implementation patterns in multiple languages
