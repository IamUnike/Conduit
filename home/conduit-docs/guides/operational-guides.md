---
icon: loader
---

# Operational Guides

### Monitoring Delivery Health

Ongoing production monitoring, as opposed to the one-time confirmation covered in Sending your first production event.

**What to monitor:**

* DLQ growth rate — a sudden increase indicates a systemic endpoint failure rather than isolated incidents.
* Delivery success rate per endpoint — trending this over time surfaces endpoints degrading gradually (e.g., a receiving service approaching a capacity limit) before they fail outright.
* Average response time per endpoint — a leading indicator of receiving-side degradation, often visible before failure rate increases.

**Programmatic monitoring via the Admin API:**

```http
import requests

response = requests.get(
    "https://api.conduit.dev/v1/deliveries",
    headers={"Authorization": f"Bearer {CONDUIT_API_KEY}"},
    params={"status": "failed", "since": "2026-07-01T00:00:00Z"}
)
 
failed_deliveries = response.json()["data"]
```

Poll this on a schedule and alert your team through existing on-call tooling when failure rates cross a threshold you define — Conduit does not currently provide native alerting/webhook-on-webhook-failure notifications; this is a common integration pattern to build on top of the Delivery Log API.

### Working with the Dead Letter Queue

**Listing DLQ entries**

```http
curl -X GET https://api.conduit.dev/v1/dead-letter-queue \
 -H "Authorization: Bearer $CONDUIT_API_KEY"
```

&#x20;**Diagnosing why a delivery landed in the DLQ**

Each DLQ entry retains its full Delivery Attempt history. Review the response\_code and response\_time\_ms of the final attempt as a starting point:

| **Signal**                            | **Likely cause**                                                                  |
| ------------------------------------- | --------------------------------------------------------------------------------- |
| Connection timeout on every attempt   | Endpoint unreachable — firewall, DNS, or downtime                                 |
| Consistent 401/403                    | Endpoint's own auth layer rejecting Conduit (not a Conduit auth issue)            |
| Consistent 404                        | Endpoint URL is wrong or has changed                                              |
| Consistent 5xx                        | Endpoint's service is erroring internally                                         |
| Mixed success/failure across attempts | Likely a capacity or intermittent-availability issue, not a hard misconfiguration |

For the full diagnostic decision tree used operationally by support teams, see Dashboard & Admin Guide → Diagnosing a failed delivery.

**Replaying from the DLQ once the issue is resolved**

```http
curl -X POST https://api.conduit.dev/v1/deliveries/{delivery_id}/replay \
  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

Replay creates a new Delivery Attempt against the existing Delivery record — it does not re-publish the original Event. See Core Concepts → Dead Letter Queue and Replay for the underlying model.

### Replaying Events and Deliveries

Two replay scopes are available, appropriate to different situations:

Replay a single delivery (retry against one specific endpoint):

```http
curl -X POST https://api.conduit.dev/v1/deliveries/{delivery_id}/replay \
  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

Replay an event to all currently subscribed endpoints (useful when an endpoint was added after the original event was published, and should receive events it missed):

```http
curl -X POST https://api.conduit.dev/v1/events/{event_id}/replay \
  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

{% hint style="warning" icon="triangle-exclamation" %}
**Caution:** Event-level replay delivers to _currently_ subscribed endpoints, which may differ from the endpoints subscribed at the time the event was originally published. Confirm current subscriptions before a broad replay to avoid delivering historical events to an endpoint that shouldn't receive them.
{% endhint %}

### Rotating Signing Secrets Safely

Covered at the security-model level in Security & Compliance → Signing secret lifecycle and rotation. This guide is the operational how-to.

**Zero-downtime rotation procedure:**

1. Generate a new secret for the endpoint. The old secret remains valid during a grace period — deliveries are not interrupted.

```
curl -X POST https://api.conduit.dev/v1/endpoints/{endpoint_id}/rotate-secret \  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

2. Deploy updated verification code on your receiving endpoint that accepts _either_ the old or new secret during the grace period (Conduit signs each delivery with the currently active secret, but your handler should be tolerant of both while the rollout is in progress across your own infrastructure, if you run multiple instances).
3. Confirm the new secret is in use by checking recent Delivery Attempts — signature verification success is visible on your end, not reportable by Conduit, since Conduit does not see your verification outcome.
4. Allow the grace period to expire. The old secret is invalidated automatically; no further action needed.

### Managing API Keys and Scoping

Every API key is scoped to exactly one Application and one Environment — a Sandbox key cannot act on Live data, and a key created under one Application cannot access another Application's resources.

**Creating a scoped key:**

```http
curl -X POST https://api.conduit.dev/v1/api-keys \
  -H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"environment": "live", "name": "billing-service-prod"}'
```

**Best practices:**

* One key per service or deployment, not one shared key across your entire infrastructure. This limits blast radius if a key is compromised and lets you revoke access to a single service without affecting others.
* Name keys descriptively (billing-service-prod, not key-1) — the name field is purely for your own organization and has no functional effect, but pays off significantly during incident response.
* Revoke unused keys. Check last\_used\_at periodically and remove keys that are no longer active.

```http
curl -X DELETE https://api.conduit.dev/v1/api-keys/{key_id} \
  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

### Setting Up Multiple Environments in CI/CD

Since Sandbox and Live require independently registered endpoints and subscriptions (see Getting Started → Sandbox vs. Live), teams running CI/CD typically script environment setup rather than configuring it by hand in the Dashboard each time.

**Example: idempotent endpoint setup script**

```python
import requests
import os
 
def ensure_endpoint(api_key: str, url: str, event_types: list[str]) -> str:
    headers = {"Authorization": f"Bearer {api_key}"}
 
    existing = requests.get(
        "https://api.conduit.dev/v1/endpoints",
        headers=headers
    ).json()["data"]
 
    match = next((e for e in existing if e["url"] == url), None)
    if match:
        requests.patch(
            f"https://api.conduit.dev/v1/endpoints/{match['id']}",
            headers=headers,
            json={"event_types": event_types}
        )
        return match["id"]
 
    created = requests.post(
        "https://api.conduit.dev/v1/endpoints",
        headers=headers,
        json={"url": url, "event_types": event_types}
    ).json()
    return created["id"]
 
# Run against Sandbox in CI, Live in the deploy pipeline
ensure_endpoint(
    api_key=os.environ["CONDUIT_API_KEY"],
    url=os.environ["WEBHOOK_URL"],
    event_types=["invoice.paid", "invoice.payment_failed"]
)
```

{% hint style="info" %}
Store separate CONDUIT\_API\_KEY values per environment in your CI/CD secrets manager, and never hardcode a Live key in a script that also runs against Sandbox by default.
{% endhint %}
