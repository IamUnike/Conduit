---
icon: bridge-suspension
---

# Advanced / Architecture Guides

### Scaling Considerations for High-Volume Event Publishing

**For integrations publishing at high sustained volume:**

* Batch where possible on your side before publishing, if your event source naturally produces bursts, Conduit's Ingest API accepts one event per request; extremely high-frequency publishing should be load-tested against your plan's rate limits (see API Reference → Rate limits).
* Design event types granularly enough that a single endpoint isn't forced to subscribe to a firehose of events it mostly discards, use filtering (see Setting up event filtering and transformation) to reduce unnecessary delivery volume rather than filtering client-side after receipt.
* Monitor DLQ growth proactively at scale, at high volume, a systemic endpoint issue generates DLQ entries fast enough that manual discovery is too slow; wire DLQ monitoring into your existing alerting (see Monitoring delivery health).

### Designing Event Schemas That Won't Need Breaking Changes

Since events are immutable once published (see [Core Concepts → Events and Event Types](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/core-concepts)), payload schema design deserves upfront care.

**Guidance:**

* Add fields, don't repurpose them. If a field's meaning needs to change, introduce a new field and deprecate the old one gradually, never change what an existing field name means, since historical events retain their original payload permanently.
* Version in the event type name only when the schema changes incompatibly (e.g., invoice.paid.v2), not for every additive change. Additive changes (new optional fields) should not require a new event type.
* Avoid embedding large or frequently-changing objects in the payload if your receiving endpoints only need a stable identifier, consider publishing an ID and letting the endpoint fetch full details via your own API, reducing payload churn and re-signing overhead on every field tweak.

### Multi-Region and Data Residency Considerations

Conduit's infrastructure spans a primary region (us-east-1) and a secondary region (eu-west-1) for EU data residency customers (see [Security & Compliance → Deployment models and data residency](security-and-compliance.md#deployment-models-and-data-residency)).

**If your Organization has EU data residency requirements:**

* Confirm your Application is provisioned in the eu-west-1 region at creation — region is not currently changeable after an Application is created.
* Delivery Log payload retention (30 days default, up to 90 on Enterprise) applies identically regardless of region.

### Using Conduit as an Internal Event Bus Across Microservices

Beyond customer-facing webhooks, some Organizations standardize _internal_ service-to-service event delivery through Conduit rather than building a separate internal message bus (see foundation use case in Section 8 of the product's internal reference).

**Pattern:**

* Each internal microservice is registered as an Endpoint, subscribed to the internal event types it cares about.
* Internal event types follow the same resource.action convention as customer-facing ones, but are typically kept in a separate Application to avoid mixing internal and external delivery logs and permissions.
* Because Conduit's model is endpoint-agnostic, it doesn't distinguish "customer" endpoints from "internal service" endpoints, this pattern requires no special configuration, only a naming and Application-organization convention your team adopts deliberately.

This pattern trades the flexibility of a purpose-built internal message bus (e.g., topic partitioning, consumer groups) for the operational simplicity of having one delivery, retry, and observability system across your entire event surface. Evaluate against your internal messaging requirements before adopting it wholesale.

#### Next Steps

* Need exact request/response schemas for any endpoint referenced above → [API Reference](api-reference.md)
* Supporting end users through delivery issues → [Dashboard & Admin Guide](dashboard-and-admin-guide.md)
* Preparing for a security review → [Security & Compliance](security-and-compliance.md)
* Something not covered here → [Troubleshooting & FAQ](troubleshooting-and-faq.md)
