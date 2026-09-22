---
icon: rectangle-api
---

# API Reference

### Authentication

All requests to the Ingest API and Admin API require an API key, sent as a Bearer token:

```
Authorization: Bearer sk_live_51H8xK2eZvKYlo2C...
```

Conduit uses API keys exclusively for API authentication, there is no OAuth flow. This is a deliberate simplicity decision: a single, unambiguous authentication mechanism keeps every code sample in this documentation identical in structure, regardless of endpoint or language.

**Key scoping**

Every API key is scoped to exactly one Application and one Environment at creation time, and that scope is fixed for the life of the key:

| **Prefix**       | **Environment** | **Notes**                                                |
| ---------------- | --------------- | -------------------------------------------------------- |
| sk\_sandbox\_... | Sandbox         | Cannot read or write Live data under any circumstance    |
| sk\_live\_...    | Live            | Cannot read or write Sandbox data under any circumstance |

A request made with a Sandbox key against Live data (or vice versa) is rejected with a 403, see Errors, below rather than silently operating on the wrong environment.

Key management is covered operationally in [Guides → Managing API keys and scoping](api-reference.md). Security properties of key storage and rotation are covered in [Security & Compliance → Authentication & Authorization Model.](security-and-compliance.md#authentication-and-authorization-model)

### Base URL, Versioning, and Conventions

**Base URL**

```
https://api.conduit.dev/v1
```

**Versioning**

The v1 path segment is the API version. Conduit does not silently change the behavior of a released API version — breaking changes are shipped under a new version segment (v2, etc.), and non-breaking changes (new optional fields, new endpoints) are added to the existing version. See API Changelog for the distinction applied in practice.

**Request and response format**

* All request bodies are JSON; requests must include Content-Type: application/json.
* All responses are JSON, including error responses.
* Timestamps are ISO 8601, UTC (2026-07-12T14:30:00Z).
* Resource identifiers are prefixed strings indicating their type (e.g., evt\_... for Events, ep\_... for Endpoints, del\_... for Deliveries), so an ID is self-describing without needing to track which endpoint returned it.

**HTTP methods**

| **Method** | **Usage**                                                         |
| ---------- | ----------------------------------------------------------------- |
| GET        | Retrieve a resource or list of resources. Never has side effects. |
| POST       | Create a resource, or trigger an action (e.g., replay).           |
| PATCH      | Partially update a resource. Omitted fields are left unchanged.   |
| DELETE     | Permanently remove a resource.                                    |

Conduit does not use PUT. Updates are always partial (PATCH) — see the note in [Guides → Registering and managing endpoints](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/guides) on how this affects fields like event\_types, which are replaced wholesale rather than merged.

### Rate Limits and Pagination

**Rate limits**

Rate limits are applied per API key and vary by plan tier:

| **Plan**   | **Requests per minute (Admin API)** | **Events per minute (Ingest API)** |
| ---------- | ----------------------------------- | ---------------------------------- |
| Free       | 60                                  | 300                                |
| Business   | 600                                 | 3,000                              |
| Enterprise | Custom                              | Custom                             |

Every response includes rate limit headers:

_X-RateLimit-Limit: 600_

_X-RateLimit-Remaining: 587_

_X-RateLimit-Reset: 1719430260_

X-RateLimit-Reset is a Unix timestamp indicating when the current window resets. When a limit is exceeded, Conduit returns 429 Too Many Requests with a Retry-After header indicating the number of seconds to wait before retrying.

Rate limit responses are not retried automatically by Conduit's delivery pipeline — this limit applies to _your calls into Conduit's API_, distinct from Conduit's own retry behavior when delivering _to your endpoints_ (**see** [Core Concepts → Retry Policies and Backoff Behavior](../core-concepts/retry-policies-and-backoff-behavior.md)).

**Pagination**

List endpoints (e.g., GET /v1/endpoints, GET /v1/deliveries) use cursor-based pagination:

```
{
"data": [ /* ... */ ],
  "has_more": true,
  "next_cursor": "cur_8f2a9c1d"
}
```

Pass next\_cursor as the cursor query parameter to retrieve the next page:

```
curl "https://api.conduit.dev/v1/deliveries?cursor=cur_8f2a9c1d&limit=50" \
-H "Authorization: Bearer $CONDUIT_API_KEY"
```

limit accepts a maximum of 100 (default 25). Do not construct URLs to pages by guessing offsets — next\_cursor is opaque and may not correspond to a simple numeric offset.

### Errors

All errors return a JSON body with a consistent shape:

```http
{
 "error": {
    "type": "invalid_request_error",
    "code": "endpoint_not_found",
    "message": "No endpoint found with id 'ep_invalid123'.",
    "request_id": "req_3f8a2c9e"
  }
}
```

Always log request\_id — it's the fastest path to resolution if you need to contact support about a specific failed request.

**HTTP status codes**

| **Status** | **Meaning**                                                                                        | **Retry-safe?**                    |
| ---------- | -------------------------------------------------------------------------------------------------- | ---------------------------------- |
| 400        | Malformed request (invalid JSON, missing required field)                                           | No — fix the request first         |
| 401        | Missing or invalid API key                                                                         | No                                 |
| 403        | Valid key, insufficient scope (e.g., Sandbox key used against Live data, or role lacks permission) | No                                 |
| 404        | Resource not found                                                                                 | No                                 |
| 409        | Conflict (e.g., duplicate resource where uniqueness is enforced)                                   | No                                 |
| 422        | Request is well-formed JSON but fails validation (e.g., invalid event\_type format)                | No                                 |
| 429        | Rate limit exceeded                                                                                | Yes — after the Retry-After window |
| 500 / 503  | Conduit-side error                                                                                 | Yes — with backoff                 |

**Common error codes**

| **code**                  | **Typical cause**                                                                          |
| ------------------------- | ------------------------------------------------------------------------------------------ |
| invalid\_api\_key         | Key is malformed, revoked, or doesn't exist                                                |
| environment\_mismatch     | Sandbox key used against a Live resource, or vice versa                                    |
| endpoint\_not\_found      | endpoint\_id doesn't exist or belongs to a different Application                           |
| event\_type\_invalid      | event\_type doesn't conform to expected format                                             |
| insufficient\_permissions | Authenticated key's role lacks permission for this action (see Core Concepts → User Roles) |
| rate\_limit\_exceeded     | Too many requests in the current window                                                    |

This is not an exhaustive list. Each resource page below lists errors specific to that resource where they diverge from this general table.

### Resources

**Events**

An Event is a discrete occurrence published into Conduit for delivery. See [Core Concepts → Events and Event Types](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/core-concepts) for the conceptual model.

**Create an event**

POST /v1/events

| **Field**   | **Type** | **Required** | **Description**                                          |
| ----------- | -------- | ------------ | -------------------------------------------------------- |
| event\_type | string   | Yes          | Category the event belongs to, e.g. invoice.paid         |
| payload     | object   | Yes          | Arbitrary JSON payload delivered to subscribed endpoints |

{% tabs %}
{% tab title="Python" %}
```python
import requests
 
response = requests.post(
    "https://api.conduit.dev/v1/events",
    headers={"Authorization": f"Bearer {CONDUIT_API_KEY}"},
    json={
        "event_type": "invoice.paid",
        "payload": {"invoice_id": "inv_12345", "amount_due": 4900, "currency": "usd"}
    }
)
```
{% endtab %}

{% tab title="cURL" %}
```http
curl -X POST https://api.conduit.dev/v1/events \
  -H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"event_type": "invoice.paid", "payload": {"invoice_id": "inv_12345", "amount_due": 4900, "currency": "usd"}}'
```
{% endtab %}

{% tab title="Node.js" %}
```javascript
const response = await fetch("https://api.conduit.dev/v1/events", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.CONDUIT_API_KEY}`,
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    event_type: "invoice.paid",
    payload: { invoice_id: "inv_12345", amount_due: 4900, currency: "usd" }
  })
});
Response — 201 Created
{
  "id": "evt_8f2a9c1d3e",
  "event_type": "invoice.paid",
  "payload": { "invoice_id": "inv_12345", "amount_due": 4900, "currency": "usd" },
  "created_at": "2026-07-12T14:30:00Z",
  "environment": "live"
}
```
{% endtab %}
{% endtabs %}

A 201 confirms Conduit accepted the event into the Event Bus — it does not confirm delivery. Check the Delivery Log or use webhooks-on-delivery-status (see Related below) to track downstream delivery outcomes.

**Retrieve an event**

```http
GET /v1/events/{id}
curl https://api.conduit.dev/v1/events/evt_8f2a9c1d3e \
  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

Returns the same object shape as the creation response.

Errors specific to this resource: 404 / event\_not\_found if the ID doesn't exist in the authenticated key's environment.

{% hint style="info" %}
Related: [Guides → Sending your first production event](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/guides) · [Guides → Designing event schemas](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/guides) that won't need breaking changes
{% endhint %}

#### Endpoints

An Endpoint is a URL registered to receive deliveries. See [Core Concepts → Endpoints and Subscriptions.](../core-concepts/endpoints-and-subscriptions.md)

**List endpoints**

```
GET /v1/endpoints
curl "https://api.conduit.dev/v1/endpoints?limit=25" \
  -H "Authorization: Bearer $CONDUIT_API_KEY"
```

Supports standard pagination (§5.3).

**Create an endpoint**

POST /v1/endpoints

| **Field**     | **Type**       | **Required** | **Description**                                       |
| ------------- | -------------- | ------------ | ----------------------------------------------------- |
| url           | string         | Yes          | Destination URL for deliveries                        |
| event\_types  | array\<string> | Yes          | Event types this endpoint subscribes to               |
| retry\_policy | object         | No           | Defaults applied if omitted — see §5.5.2 schema below |

```python
response = requests.post(
"https://api.conduit.dev/v1/endpoints",
    headers={"Authorization": f"Bearer {CONDUIT_API_KEY}"},
    json={
        "url": "https://your-service.example.com/webhooks",
        "event_types": ["invoice.paid"]
    }
)
// Response — 201 Created
{
  "id": "ep_4b7f1a9d",
  "url": "https://your-service.example.com/webhooks",
  "event_types": ["invoice.paid"],
  "signing_secret": "whsec_a1b2c3d4e5f6...",
  "retry_policy": {
    "max_attempts": 6,
    "initial_delay_seconds": 30,
    "backoff_multiplier": 2,
    "max_delay_seconds": 7200
  },
  "status": "active",
  "filters": []
}
```

signing\_secret is returned only in this creation response. It is not retrievable afterward — store it immediately. See [Security & Compliance → Signing secret lifecycle and rotation](security-and-compliance.md#signing-secret-lifecycle-and-rotation) if it's lost (rotation, not retrieval, is the recovery path).

**Retrieve, update, or delete an endpoint**

GET    /v1/endpoints/{id}

PATCH  /v1/endpoints/{id}

DELETE /v1/endpoints/{id}

PATCH accepts any subset of url, event\_types, retry\_policy, filters, or status. Fields omitted from the request body are left unchanged; array fields (event\_types, filters) are replaced, not merged — see Guides → Subscribing to event types.

```http
curl -X PATCH https://api.conduit.dev/v1/endpoints/ep_4b7f1a9d \
-H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "paused"}'
```

Errors specific to this resource: 422 / invalid\_url if url isn't a valid HTTPS URL (HTTP-only URLs are rejected — see [Security & Compliance → Security overview](security-and-compliance.md#security-overview)).

#### Deliveries

A Delivery represents one event's delivery lifecycle to one specific endpoint. See Core Concepts → Deliveries and Delivery Attempts.

**List deliveries**

GET /v1/deliveries

| **Query parameter** | **Description**                              |
| ------------------- | -------------------------------------------- |
| event\_id           | Filter to deliveries for a specific event    |
| endpoint\_id        | Filter to deliveries for a specific endpoint |
| status              | pending \| succeeded \| failed \| exhausted  |
| since / until       | ISO 8601 range filter on created\_at         |

```http
curl "https://api.conduit.dev/v1/deliveries?status=failed&since=2026-07-01T00:00:00Z" \
 -H "Authorization: Bearer $CONDUIT_API_KEY"
```

Response

```json
{
"data": [
    {
      "id": "del_9c3e7b2a",
      "event_id": "evt_8f2a9c1d3e",
      "endpoint_id": "ep_4b7f1a9d",
      "status": "failed",
      "attempts": [
        {
          "id": "att_1a2b3c",
          "attempt_number": 1,
          "response_code": 503,
          "response_time_ms": 4021,
          "attempted_at": "2026-07-12T14:30:05Z"
        }
      ]
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

&#x20;**Replay a delivery**

POST /v1/deliveries/{id}/replay

```http
curl -X POST https://api.conduit.dev/v1/deliveries/del_9c3e7b2a/replay \
-H "Authorization: Bearer $CONDUIT_API_KEY"
```

Creates a new Delivery Attempt against the existing Delivery record. Does not create a new Event. See [Core Concepts → Dead Letter Queue and Replay.](../core-concepts/dead-letter-queue-and-replay.md)

Errors specific to this resource: 409 / replay\_in\_progress if a replay of this delivery is already in flight.

#### Dead Letter Queue

GET /v1/dead-letter-queue

Returns Deliveries with status: exhausted, in the same shape as GET /v1/deliveries, filtered server-side. Accepts the same endpoint\_id, since, and until query parameters as §5.5.3.

```http
curl "https://api.conduit.dev/v1/dead-letter-queue?endpoint_id=ep_4b7f1a9d" \
-H "Authorization: Bearer $CONDUIT_API_KEY"
```

This endpoint is a filtered read view, not a separate resource type — a DLQ entry and a Delivery with status: exhausted are the same underlying record. Replay uses the same POST /v1/deliveries/{id}/replay call described in §5.5.3.

Related: Guides → Working with the Dead Letter Queue

#### API Keys

POST   /v1/api-keys

DELETE /v1/api-keys/{id}

**Create a key**

| **Field**   | **Type** | **Required** | **Description**                                           |
| ----------- | -------- | ------------ | --------------------------------------------------------- |
| environment | string   | Yes          | sandbox or live                                           |
| name        | string   | No           | Human-readable label; recommended for operational clarity |

```http
curl -X POST https://api.conduit.dev/v1/api-keys \
-H "Authorization: Bearer $CONDUIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"environment": "live", "name": "billing-service-prod"}'

```

&#x20; **Response — 201 Created**

```json
{
"id": "key_2d8f1e9a",
  "prefix": "sk_live_51H8xK2e...",
  "environment": "live",
  "name": "billing-service-prod",
  "created_at": "2026-07-12T14:30:00Z",
  "last_used_at": null
}
```

The full key value is returned once, at creation, and is not retrievable afterward — only the prefix is stored and returned on subsequent reads, sufficient for identification but not for authentication.

**Delete (revoke) a key — immediate and irreversible:**

```http
curl -X DELETE https://api.conduit.dev/v1/api-keys/key_2d8f1e9a \
 -H "Authorization: Bearer $CONDUIT_API_KEY"
```

#### Applications

GET /v1/applications/{id}

Read-only via the Admin API in the current version — Application creation and deletion are performed through the Dashboard only (see Dashboard & Admin Guide → Managing Applications and Environments).

```http
curl https://api.conduit.dev/v1/applications/app_7e2c1b9f \
-H "Authorization: Bearer $CONDUIT_API_KEY"
Response
{
  "id": "app_7e2c1b9f",
  "name": "My First Integration",
  "created_at": "2026-06-01T09:15:00Z"
}
```

### Webhooks Reference: What Conduit Sends

<mark style="color:$info;">This section documents the exact structure of a Conduit delivery — the request your endpoint receives — as opposed to the requests you send</mark> <mark style="color:$info;"></mark>_<mark style="color:$info;">to</mark>_ <mark style="color:$info;"></mark><mark style="color:$info;">Conduit's API, documented above.</mark>

**Request headers**

| **Header**          | **Description**                                                             |
| ------------------- | --------------------------------------------------------------------------- |
| Content-Type        | Always application/json                                                     |
| Conduit-Signature   | HMAC-SHA256 signature; see below                                            |
| Conduit-Timestamp   | Unix timestamp used in signature computation                                |
| Conduit-Event-Id    | Same value as the payload's event\_id, provided as a header for convenience |
| Conduit-Delivery-Id | The Delivery record this attempt belongs to                                 |

**Request body**

```json
{
"event_id": "evt_8f2a9c1d3e",
  "event_type": "invoice.paid",
  "created_at": "2026-07-12T14:30:00Z",
  "payload": {
    "invoice_id": "inv_12345",
    "amount_due": 4900,
    "currency": "usd"
  }
}
```

**Signature verification**

The Conduit-Signature header is computed as described in Core Concepts → Payload Signing (see FIG-DIAG-02) and implemented per-language in Guides → Verifying webhook signatures. Full byte-level and replay-prevention detail is in Security & Compliance → Payload signing and verification deep dive.

**Expected response**

Your endpoint must respond with a 2xx status code within your endpoint's configured timeout to be counted as a successful delivery. Response _bodies_ are logged but not otherwise interpreted by Conduit — return an empty 200 unless you have your own reason to include a body.

### API Changelog

Schema-level changes to the API are tracked separately from the product-wide Changelog (§10), since not every product change affects the API surface.

Format:

`## 2026-07-01`

`### Added`

\- \`filters\` field on Endpoint (non-breaking)

`## 2026-05-15`

`### Changed`

\- \`GET /v1/deliveries\` pagination switched from offset- to cursor-based (breaking — see migration note)

Breaking changes are never introduced into an existing version (v1) — see Base URL, Versioning, and Conventions above. Entries marked "breaking" in this changelog apply only to the next major version and include a migration guide link.

***

#### Next Steps

* Need the conceptual model behind these resources → [Core Concepts](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/core-concepts)
* Task-oriented walkthroughs using these endpoints → [Guides](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/guides)
* Operating the Dashboard rather than the API directly → [Dashboard & Admin Guide](dashboard-and-admin-guide.md)

<br>

&#x20;
