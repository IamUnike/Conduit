---
description: >-
  This section is the conceptual backbone of the Conduit documentation suite.
  Every Guide, API page, and Dashboard workflow assumes the model described
  here.
icon: book
---

# How Conduit Works

At a high level, Conduit sits between your backend and the endpoints you need to notify. You publish events; Conduit takes responsibility for getting them delivered.

**The pipeline has six stages:**

1. **Ingest:** Your backend sends an event to Conduit via a single API call (POST /v1/events). Conduit acknowledges receipt immediately; it does not wait for delivery to complete before responding.
2. **Event Bus:** The event is written to a durable, ordered queue. This decouples "Conduit accepted the event" from "Conduit delivered the event" — if every downstream endpoint were unreachable right now, the event would still be safely queued.
3. **Subscription Matching:** Conduit determines which registered endpoints are subscribed to this event's type.
4. **Delivery:** For each matched endpoint, a Delivery Worker signs the payload and sends it over HTTPS. If the endpoint doesn't respond with a success status, the worker retries according to that endpoint's retry policy.
5. **Logging:** Every delivery attempt — successful or not — is recorded with its response code, response time, and body.
6. **Dead Letter Queue:** If all retries are exhausted without success, the delivery moves to the Dead Letter Queue, where it can be inspected and manually replayed once the underlying issue is fixed.

The Dashboard and Admin API are not part of this pipeline — they read from and write to the same underlying data (Applications, Endpoints, Delivery Logs) but sit outside the delivery path. A Dashboard outage, for example, would never affect whether an event gets delivered.

{% hint style="info" %}
Each of the concepts named above — Event, Endpoint, Subscription, Delivery, Delivery Attempt, Retry Policy, Dead Letter Queue — is defined in detail in the sections that follow.
{% endhint %}
