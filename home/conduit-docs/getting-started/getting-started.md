---
icon: rocket-launch
---

# Getting Started

### What is Conduit?

Conduit is a managed platform for delivering outbound webhooks. If your application needs to notify external systems, your own customers' endpoints, internal services, or third-party integrations — when something happens (an invoice is paid, a user is created, a shipment status changes), Conduit handles the delivery of that notification.

You send Conduit a single event through the Ingest API. Conduit takes over from there: it identifies which endpoints are subscribed to that event type, signs the payload, delivers it over HTTPS, retries automatically on failure, and records the full delivery history so you can answer "did this webhook go through?" without guessing.

<figure><img src="../.gitbook/assets/ChatGPT Image Jul 13, 2026, 05_30_59 PM.png" alt=""><figcaption></figcaption></figure>

Conduit replaces the retry loops, signature verification, and delivery logging that most teams end up building, and rebuilding in-house.

