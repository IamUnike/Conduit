---
icon: square-dashed
---

# Dead Letter Queue and Replay

The Dead Letter Queue (DLQ) holds deliveries that have exhausted every retry attempt in their endpoint's retry policy without receiving a success response. A delivery in the DLQ is not lost, its full attempt history remains available, and it can be replayed once the underlying issue (an outage, a misconfiguration, an expired certificate) is resolved.

Replay is the act of manually re-triggering delivery of a past event. Replay can be performed:

* On a single delivery, from the DLQ or from the general Delivery Log
* On an entire event, resending it to all currently subscribed endpoints, useful if you want a delivery retried against an endpoint that was added _after_ the original event was published

Replaying a delivery creates a new Delivery Attempt against the existing Delivery record; it does not create a new Event. This preserves a complete, accurate history of every attempt, original and replayed, against that event.

**When to replay vs. when to let retries run their course:** The default retry policy is designed to handle transient failures (brief outages, deploys, momentary network issues) automatically. Manual replay exists for cases where the failure required human intervention to fix, a wrong URL, an expired TLS certificate, a firewall rule, where the automated retry window closed before the fix was made.

{% hint style="info" %}
For hands-on replay procedures, see [Dashboard & Admin Guide → Replaying deliveries from the Dashboard](../guides/dashboard-and-admin-guide.md#replaying-deliveries-from-the-dashboard) and [Guides → Working with the Dead Letter Queue.](../guides/operational-guides.md#working-with-the-dead-letter-queue)
{% endhint %}
