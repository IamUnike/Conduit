---
icon: square-question
---

# Troubleshooting & FAQ

### "My Webhook Didn't Arrive" — Diagnostic Flowchart

This is the single most common support entry point. Work through the checks below in order — each rules out a specific, common cause before moving to the next.

**Step-by-step:**

1. Confirm the event was actually published. Search Events by approximate timestamp in the Dashboard, or query GET /v1/events/{id} if you have the ID. If no matching event exists, the issue is upstream of Conduit — check the publishing service's own logs.
2. Confirm a Delivery record exists for that event and the relevant endpoint. Filter Deliveries by event\_id and endpoint\_id. If no Delivery record exists at all, the endpoint was very likely not subscribed to that event type at the time the event was published — see Core Concepts → Endpoints and Subscriptions.
3. If a Delivery record exists, proceed to Diagnosing a Failed Delivery (Dashboard & Admin Guide → §7.5, FIG-DIAG-04) to determine the specific failure cause.
4. If the Delivery shows succeeded, the issue is downstream of Conduit — confirm the receiving application actually processed the payload rather than merely acknowledging receipt. A 200 response confirms Conduit's delivery succeeded; it does not confirm the receiving application's internal processing succeeded.

### Common Error Codes and What They Mean

This table consolidates error codes referenced individually throughout the API Reference and Guides into one lookup page.

| **Error code**            | **HTTP status** | **Meaning**                                                                   | **Where to look next**                                                                              |
| ------------------------- | --------------- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| invalid\_api\_key         | 401             | Key is malformed, revoked, or doesn't exist                                   | Confirm the key was copied in full and hasn't been deleted (Guides → Managing API keys and scoping) |
| environment\_mismatch     | 403             | Sandbox key used against Live data, or vice versa                             | §9.5 below                                                                                          |
| insufficient\_permissions | 403             | Authenticated key's role lacks permission for this action                     | Dashboard & Admin Guide → Team management and role assignment                                       |
| endpoint\_not\_found      | 404             | endpoint\_id doesn't exist, or belongs to a different Application/Environment | Confirm the ID and that you're querying the correct environment                                     |
| event\_type\_invalid      | 422             | event\_type doesn't conform to expected format                                | Use lowercase, dot-separated resource.action format                                                 |
| invalid\_url              | 422             | Endpoint URL is missing, malformed, or not HTTPS                              | Security & Compliance → Network security                                                            |
| rate\_limit\_exceeded     | 429             | Too many requests in the current window                                       | §9.6 below                                                                                          |
| replay\_in\_progress      | 409             | A replay of this delivery is already in flight                                | Wait for the in-flight replay to complete before retrying                                           |

For the complete, authoritative error reference, see API Reference → Errors.

### Signature Verification Failures

This is the most common integration-level bug reported by teams implementing Conduit, and warrants its own page given its frequency.

**Checklist, in order of likelihood:**

1. Are you verifying against the raw request body? If your web framework parses the JSON body before your verification code runs, and you re-serialize it to compute the signature, the byte sequence will differ from what Conduit actually signed — even if the data is identical. Read the raw body _before_ any JSON parsing middleware touches it. See Security & Compliance → Payload Signing and Verification: Deep Dive for why this matters at the byte level.
2. Are you using the correct signing secret for the environment? Sandbox and Live endpoints have different signing secrets. Verifying a Live delivery against a Sandbox secret (or vice versa) will always fail. This is the single most common cause of "verification worked in testing, fails in production."
3. Has the signing secret been rotated recently? During a rotation grace period, deliveries may be signed with either the old or new secret (Security & Compliance → Signing Secret Lifecycle and Rotation). Verification code that checks against only one secret will intermittently fail during rotation.
4. Are you using constant-time comparison? A functional bug, not a security bug, but worth ruling out: some naive implementations using standard == string comparison can behave inconsistently under certain framework/language combinations. Use the language-specific constant-time comparison shown in Guides → Verifying webhook signatures.
5. Is your timestamp tolerance window rejecting valid deliveries? If your server's clock is meaningfully out of sync with UTC, a legitimately fresh delivery may fall outside your tolerance window and be rejected as if it were stale. Confirm NTP synchronization on the receiving server.

If all five checks pass and verification still fails, capture the exact raw payload bytes, the Conduit-Timestamp and Conduit-Signature header values, and the request\_id from a failed Delivery Attempt, and contact support (§9.8) — this is enough information to reproduce the signature computation independently.

### Endpoint Stuck in Paused State

**Symptom:** An endpoint shows status: paused and isn't receiving deliveries, but no one recalls pausing it.

**Common causes:**

* A team member paused it intentionally during maintenance and forgot to resume it — check the endpoint's activity history for who changed its status and when.
* A CI/CD script overwrote the status field. If your deployment pipeline uses PATCH /v1/endpoints/{id} to update other fields (e.g., event\_types) and includes a stale status: paused value from a template or previous configuration, it will silently re-pause the endpoint on every deploy. Review Guides → Setting up multiple environments in CI/CD and confirm your script only includes fields it intends to change.

**Resolution:** Toggle the endpoint back to active from the Dashboard or via PATCH /v1/endpoints/{id} with {"status": "active"}. Remember that resuming does not automatically retry deliveries that were skipped while paused — replay those manually if needed (Guides → Working with the Dead Letter Queue).

### Sandbox vs. Live Confusion

**Symptom:** environment\_mismatch errors, or a delivery that "should have happened" doesn't appear anywhere in the Dashboard.

This almost always traces to one of two causes:

1. Wrong API key. Confirm the key prefix (sk\_sandbox\_... vs. sk\_live\_...) matches the environment you intend to operate in. This is especially common when a Sandbox key is accidentally left in a production deployment's environment variables after initial testing.
2. Wrong environment selected in the Dashboard. The environment selector (top navigation) persists across sessions — if a Live delivery seems to be missing, confirm the Dashboard isn't still showing Sandbox from a previous session.

{% hint style="warning" %}
Prevention: Name your API keys descriptively at creation (e.g., billing-service-prod-live, not key-1) so environment mismatches are visible at a glance in your secrets manager — see Guides → Managing API keys and scoping.
{% endhint %}

### Rate Limit Errors

**Symptom:** 429 Too Many Requests / rate\_limit\_exceeded.

**Resolution:**

1. Check the Retry-After header on the 429 response — this tells you exactly how many seconds to wait before retrying.
2. Check X-RateLimit-Limit on recent successful responses to confirm your current plan's limit (API Reference → Rate Limits and Pagination).
3. If you're consistently near the limit under normal operation (not a traffic spike), consider whether request batching is possible on your end, or contact your account representative about an Enterprise plan with custom limits.

This limit applies to your calls into Conduit's API — it is unrelated to Conduit's own delivery retry behavior when sending webhooks to _your_ endpoints. Don't confuse a 429 from the Ingest/Admin API with a delivery retry, which is governed entirely by the endpoint's retry policy (Core Concepts → Retry Policies and Backoff Behavior).

### Frequently Asked Questions

<details>

<summary>Does Conduit offer a self-hosted or on-premises deployment option? </summary>

No. Conduit is offered exclusively as a cloud service — multi-tenant managed cloud (default) or single-tenant dedicated cloud (Enterprise). This is a deliberate product decision, not a current limitation expected to change. See Security & Compliance → Deployment Models and Data Residency.

</details>

<details>

<summary>How long is delivery data retained? </summary>

Delivery Log payload bodies are retained 30 days by default, up to 90 days on the Enterprise plan. See Security & Compliance → Data Retention and Deletion Policy.

</details>

<details>

<summary>Can I get exactly-once delivery guarantees? </summary>

No. Conduit guarantees at-least-once delivery. Your receiving endpoint should be idempotent using the event\_id field. See Core Concepts → Delivery Guarantees and Guides → Handling idempotency on the receiving end.

</details>

<details>

<summary>Is there a staging environment separate from Sandbox and Live?</summary>

No — Conduit supports exactly two environments per Application, by design. See Core Concepts → Organizations, Applications, and Environments.

</details>

<details>

<summary>Can I use RSA or another asymmetric signing scheme instead of HMAC-SHA256?</summary>

No, HMAC-SHA256 is the only supported signing scheme in the current version. See Security & Compliance → Payload Signing and Verification: Deep Dive.

</details>

<details>

<summary>What happens to my Delivery Logs if I delete an endpoint?</summary>

Historical Delivery Log entries are retained (subject to the standard retention window) even after the endpoint itself is deleted. See Guides → Registering and managing endpoints.

</details>

<details>

<summary>How is billing calculated? </summary>

Usage-based, on delivered events per month, aggregated at the Organization level across all Applications. Sandbox activity is never billed. See Dashboard & Admin Guide → Billing and Plan Management.

</details>

<details>

<summary>Can a single event be delivered to more than one endpoint? </summary>

Yes — if multiple endpoints are subscribed to the same event type, Conduit creates one independent Delivery per endpoint. See Core Concepts → Deliveries and Delivery Attempts.

</details>

### Contacting Support and Escalation Path

Self-service first: Most issues are resolvable using §9.1–9.7 above, or the diagnostic tools in Dashboard & Admin Guide → Diagnosing a Failed Delivery.

**When to contact support directly:**

* A suspected Conduit-side issue (elevated 5xx responses from Conduit's own API, not from your endpoint)
* Billing or plan discrepancies
* Security concerns (route to security@latticesystems.example instead — see Security & Compliance → Responsible Disclosure)
* Any issue where the diagnostic steps above were followed completely and the cause remains unclear

**What to include in a support request:**

* The request\_id from the relevant API response, or the Delivery/Event ID in question
* The environment (Sandbox or Live) and Application affected
* A summary of the diagnostic steps already taken from this page

Support request volume and response time targets vary by plan tier — Enterprise customers should refer to their support SLA in their account agreement rather than the general targets published on the support portal.

***

#### Next Steps

This is typically the last stop in an active troubleshooting session. From here:

* Confirmed root cause, need to change configuration → Guides
* Need exact schema/error detail for a specific endpoint → API Reference
* Issue was resolved, want to prevent recurrence → revisit the relevant Core Concepts page for the underlying model
