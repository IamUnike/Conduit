---
description: >-
  Each guide assumes familiarity with Core Concepts and links back to it rather
  than re-explaining terms.
icon: graduation-cap
---

# Integration Guides

### **Registering and Managing Endpoints**

This guide covers endpoint lifecycle operations beyond the initial registration shown in Getting Started: updating an endpoint's configuration, pausing and resuming delivery, and removing an endpoint safely.

**Updating an endpoint**

You can update an endpoint's URL, event type subscriptions, or retry policy without changing its id or signing secret. Existing Deliveries and Delivery Attempts are unaffected by the update — history is preserved.

{% tabs %}
{% tab title="Python" %}
```python
import requests
 
response = requests.patch(
    f"https://api.conduit.dev/v1/endpoints/{endpoint_id}",
    headers={"Authorization": f"Bearer {CONDUIT_API_KEY}"},
    json={"url": "https://your-service.example.com/webhooks/v2"}
)
```
{% endtab %}

{% tab title="cURL" %}
```hurl
curl -X PATCH https://api.conduit.dev/v1/endpoints/{endpoint_id} \
  -H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://your-service.example.com/webhooks/v2"}'
```
{% endtab %}

{% tab title="Node.js" %}
```javascript
const response = await fetch(`https://api.conduit.dev/v1/endpoints/${endpointId}`, {
  method: "PATCH",
  headers: {
    "Authorization": `Bearer ${process.env.CONDUIT_API_KEY}`,
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ url: "https://your-service.example.com/webhooks/v2" })
});
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
**Note:** Changing an endpoint's URL takes effect immediately for all future delivery attempts, including retries already in progress for existing Deliveries. If an in-flight retry is expected to hit the _old_ URL, pause the endpoint first, let outstanding retries exhaust or replay them manually afterward, then update the URL.
{% endhint %}

**Pausing and resuming an endpoin**t

Pausing stops an endpoint from receiving new deliveries while preserving its configuration and history — useful during planned maintenance on the receiving system.

```hurl
curl -X PATCH https://api.conduit.dev/v1/endpoints/{endpoint_id} \
  -H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "paused"}'
```

While paused, matched events still create Delivery records, but no attempts are made against the endpoint until it's set back to active. This means resuming an endpoint does not automatically retry everything that was queued during the pause — you'll typically want to replay affected deliveries manually afterward. See Working with the Dead Letter Queue below.

**Deleting an endpoint**

Deletion is permanent and cannot be undone. Deleting an endpoint does not delete its historical Delivery Log entries — those remain queryable for audit purposes, subject to your Organization's data retention policy (see Security & Compliance → Data retention and deletion policy).

```hurl
curl -X DELETE https://api.conduit.dev/v1/endpoints/{endpoint_id} \
  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

{% hint style="warning" icon="triangle-exclamation" %}
Before deleting: Consider pausing instead, if there's any chance the endpoint will be reused. Deletion is intended for endpoints that are permanently retired.
{% endhint %}

### Subscribing to Event Types

Subscriptions determine which events an endpoint receives. An endpoint with zero subscriptions is registered but functionally inert — it will never receive a delivery.

**Adding a subscription to an existing endpoint**

```javascript
import requests

response = requests.patch(
    f"https://api.conduit.dev/v1/endpoints/{endpoint_id}",
    headers={"Authorization": f"Bearer {CONDUIT_API_KEY}"},
    json={"event_types": ["invoice.paid", "invoice.payment_failed"]}
)
```

{% hint style="info" %}
Important: The event\_types field is set, not appended — submitting \["invoice.payment\_failed"] on an endpoint currently subscribed to \["invoice.paid"] replaces the subscription list rather than adding to it. Always include the full desired set of event types in the request.
{% endhint %}

**Design guidance: one endpoint vs. multiple**

Two common patterns:

| **Pattern**                             | **When to use it**                                                                                                                                                                                                   |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| One endpoint, many event types          | Simpler operationally; your receiving service inspects event\_type in the payload and routes internally. Recommended default.                                                                                        |
| Multiple endpoints, one event type each | Useful when different event types are handled by genuinely different services, or when you want independent retry policies and delivery logs per event category (e.g., billing events vs. account lifecycle events). |

There is no limit on the number of endpoints per Application, so the choice is architectural rather than a platform constraint.

### Sending Your First Production Event

This guide extends the Sandbox flow from Getting Started into Live, and covers the operational details that matter once real traffic is flowing.

**Checklist before your first Live event:**

* [ ] Live endpoint registered, with the correct production URL
* [ ] Live signing secret stored securely and wired into your verification code (distinct from the Sandbox secret — see Sandbox vs. Live)
* [ ] Receiving handler deployed and reachable from Conduit's published egress IP ranges (see Security & Compliance → Network security)
* [ ] Receiving handler is idempotent (see Handling idempotency on the receiving end, below)
* [ ] Retry policy reviewed and intentionally configured, not left at defaults if your use case has specific latency or aggressiveness requirements

**Publishing the event**

The request shape is identical to Sandbox — only the API key differs.

```hurl
curl -X POST https://api.conduit.dev/v1/events \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "invoice.paid",
    "payload": {
      "invoice_id": "inv_9f21ac",
      "amount_due": 12000,
      "currency": "usd"
    }
  }'
```

**Confirming delivery in production**

Check the Delivery Log filtered by environment=live rather than relying on the Dashboard's default view, which may default to your most recently active environment.

For ongoing production monitoring rather than a one-time check, see Monitoring delivery health below.

### Handling Idempotency on the Receiving End

Conduit guarantees at-least-once delivery (see Core Concepts → Delivery Guarantees), which means your receiving endpoint may occasionally receive the same delivery more than once. This guide shows how to make your handler safe against duplicates.

**The pattern**

Every event carries a stable event\_id. Before processing a delivery, check whether that event\_id has already been processed; if so, acknowledge success without repeating the side effect.

{% tabs %}
{% tab title="Python" %}
```python
import redis
from flask import Flask, request
 
app = Flask(__name__)
r = redis.Redis()
 
@app.route("/webhooks", methods=["POST"])
def handle_webhook():
    event = request.get_json()
    event_id = event["event_id"]
 
    # SETNX returns False if the key already exists — i.e., a duplicate
    is_new = r.setnx(f"processed:{event_id}", 1)
    if not is_new:
        return "", 200  # Acknowledge duplicate without reprocessing
 
    r.expire(f"processed:{event_id}", 60 * 60 * 24 * 7)  # 7-day dedup window
    process_invoice_paid(event["payload"])
    return "", 200
```
{% endtab %}

{% tab title="Node,js" %}
```hurl
app.post("/webhooks", async (req, res) => {
  const { event_id, payload } = req.body;
 
  const isNew = await redis.setnx(`processed:${event_id}`, 1);
  if (!isNew) {
    return res.status(200).send(); // Duplicate — acknowledge, don't reprocess
  }
 
  await redis.expire(`processed:${event_id}`, 60 * 60 * 24 * 7);
  await processInvoicePaid(payload);
  res.status(200).send();
});
```
{% endtab %}
{% endtabs %}

**Design notes:**

* Dedup window length is a judgment call. Seven days comfortably exceeds any realistic retry window under the default retry policy, but confirm this against your endpoint's actual configured policy.
* Respond 200 for duplicates, not an error status. Returning an error for a duplicate causes Conduit to retry it again, which is both unnecessary and wasteful.
* Don't rely on the receiving order of deliveries as a substitute for idempotency — Conduit does not guarantee strict in-order delivery across retries and concurrent endpoints.

### Verifying Webhook Signatures

Full conceptual background is in [Core Concepts → Payload Signing](../core-concepts/payload-signing.md) and [Security & Compliance → Payload signing and verification deep dive](security-and-compliance.md#payload-signing-and-verification-deep-dive). This guide is the copy-pasteable, per-language implementation reference.

{% tabs %}
{% tab title="Python" %}
```python
import hmac
import hashlib
import time
 
def verify_signature(payload_body: bytes, timestamp: str, signature: str, secret: str, tolerance_seconds: int = 300) -> bool:
    # Reject stale timestamps to mitigate replay attacks
    if abs(time.time() - int(timestamp)) > tolerance_seconds:
        return False
 
    signed_content = f"{timestamp}.{payload_body.decode()}"
    expected_signature = hmac.new(
        secret.encode(),
        signed_content.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected_signature, signature)
```
{% endtab %}

{% tab title="Node.js" %}
```javascript
const crypto = require("crypto");
 
function verifySignature(payloadBody, timestamp, signature, secret, toleranceSeconds = 300) {
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > toleranceSeconds) {
    return false; // Stale timestamp — reject
  }
 
  const signedContent = `${timestamp}.${payloadBody}`;
  const expectedSignature = crypto
    .createHmac("sha256", secret)
    .update(signedContent)
    .digest("hex");
 
  return crypto.timingSafeEqual(
    Buffer.from(expectedSignature),
    Buffer.from(signature)
  );
}
```
{% endtab %}

{% tab title="Go" %}
```go
func VerifySignature(payloadBody []byte, timestamp, signature, secret string, toleranceSeconds int64) bool {
    ts, err := strconv.ParseInt(timestamp, 10, 64)
    if err != nil || abs(time.Now().Unix()-ts) > toleranceSeconds {
        return false
    }
 
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write([]byte(timestamp + "." + string(payloadBody)))
    expectedSignature := hex.EncodeToString(mac.Sum(nil))
 
    return hmac.Equal([]byte(expectedSignature), []byte(signature))
}
```
{% endtab %}

{% tab title="Ruby" %}
```ruby
require "openssl"
 
def verify_signature(payload_body, timestamp, signature, secret, tolerance_seconds: 300)
  return false if (Time.now.to_i - timestamp.to_i).abs > tolerance_seconds
 
  signed_content = "#{timestamp}.#{payload_body}"
  expected_signature = OpenSSL::HMAC.hexdigest("SHA256", secret, signed_content)
 
  OpenSSL.fixed_length_secure_compare(expected_signature, signature)
end
```
{% endtab %}
{% endtabs %}

**Critical implementation notes:**

* Use constant-time comparison (hmac.compare\_digest, crypto.timingSafeEqual, hmac.Equal, fixed\_length\_secure\_compare). A naive == comparison is vulnerable to timing attacks that can leak the correct signature byte-by-byte.
* Verify against the raw request body, not a re-serialized version of the parsed JSON. Re-serializing can change key ordering or whitespace, producing a payload that no longer matches the signature Conduit computed. Read the raw body _before_ your framework parses it.
* Enforce a timestamp tolerance window to reject replayed deliveries — an attacker who intercepts a valid signed payload should not be able to resend it indefinitely.

Common failure modes are cataloged in [Troubleshooting → Signature verification failures.](troubleshooting-and-faq.md#signature-verification-failures)

### Configuring Retry Policies

Retry policies are set per endpoint. Configurable parameters:

| **Parameter**           | **Description**                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------ |
| max\_attempts           | Total number of delivery attempts before moving to the Dead Letter Queue             |
| initial\_delay\_seconds | Delay before the first retry (after the initial attempt fails)                       |
| backoff\_multiplier     | Factor by which the delay increases after each failed attempt                        |
| max\_delay\_seconds     | Ceiling on the delay between attempts, regardless of how many failures have occurred |

Example: a more aggressive policy for a latency-sensitive endpoint

```hurl
curl -X PATCH https://api.conduit.dev/v1/endpoints/{endpoint_id} \
  -H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "retry_policy": {
      "max_attempts": 4,
      "initial_delay_seconds": 10,
      "backoff_multiplier": 3,
      "max_delay_seconds": 600
    }
  }'
```

**Choosing values for your use case:**

* High-availability endpoints behind auto-scaling infrastructure can tolerate a shorter overall retry window — failures are more likely to be genuine misconfigurations than transient capacity issues.
* Endpoints owned by smaller or less operationally mature teams benefit from a longer overall window (more attempts, higher max\_delay\_seconds) to absorb occasional extended downtime without events reaching the DLQ prematurely.
* Review [Core Concepts → Retry Policies](../core-concepts/retry-policies-and-backoff-behavior.md) and Backoff Behavior for how these parameters combine to produce the actual delay schedule.

### Setting Up Event Filtering and Transformation

Event filtering lets an endpoint receive only a subset of events matching its subscribed event types, based on payload field values — useful when an endpoint only cares about a fraction of a broad event category.

Example: only deliver invoice.paid events above a threshold amount

```hurl
curl -X PATCH https://api.conduit.dev/v1/endpoints/{endpoint_id} \
-H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "filters": [
      { "field": "payload.amount_due", "operator": "greater_than", "value": 10000 }
    ]
  }'
```

Events that don't match an endpoint's filters are not delivered to that endpoint and do not count against its delivery log — they're excluded before the Delivery record is created, not delivered-and-discarded.

Filtering vs. subscriptions: Subscriptions determine _which event types_ an endpoint can receive at all. Filters narrow that further, based on payload content. Use subscriptions for coarse routing and filters for fine-grained conditions within a type.

### Migrating from an In-House Webhook System to Conduit

Most in-house webhook systems evolved reactively (see Core Concepts → How Conduit Works, and the foundation problem statement it's built from) and rarely have a clean 1:1 mapping to Conduit's model. This guide outlines a low-risk migration path.

**Recommended sequence:**

1. Inventory your existing event types and endpoints. Map each in-house event name to a Conduit event\_type using the resource.action convention (see [Core Concepts → Events and Event Types](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/core-concepts)).
2. Stand up Conduit in parallel, not as a replacement, initially. Publish events to both your legacy system and Conduit simultaneously. Register your existing customer endpoints in Conduit's Sandbox first, then Live, without yet cutting over delivery.
3. Validate against production traffic without customer impact. Compare Conduit's delivery outcomes against your legacy system's for the same events, using Conduit's Delivery Log as the source of truth for response codes and timing.
4. Cut over one endpoint or customer segment at a time. Disable legacy delivery for that segment once Conduit's behavior is validated, rather than a single global cutover.
5. Decommission the legacy system once all segments are migrated and a full retry-cycle window has passed with no discrepancies.

**Common migration gotchas:**

* Legacy systems that retried indefinitely (no backoff ceiling) may have trained downstream teams to expect eventual delivery no matter how long an endpoint was down. Conduit's finite retry policy plus DLQ requires communicating a different operational model to those teams.
* If your legacy system didn't sign payloads, your customers' receiving endpoints won't have verification logic yet. Budget migration time for customer-facing communication, not just your own infrastructure change.
