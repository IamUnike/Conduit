---
icon: truck-ramp
---

# Deliveries and Delivery Attempts

A Delivery represents one event's delivery lifecycle to one specific endpoint. If an event matches three subscribed endpoints, Conduit creates three separate Deliveries, one per endpoint each tracked independently.

**A Delivery has a status:**

* pending: queued, not yet attempted or currently retrying
* succeeded: an attempt returned a success response
* failed: the most recent attempt failed, but retries remain
* exhausted: all retries have been used without success; the delivery has moved to the Dead Letter Queue

Each Delivery contains one or more Delivery Attempts. A Delivery Attempt is a single HTTP request made as part of that delivery's retry sequence. Every attempt is logged with its response\_code, response\_time\_ms, and attempted\_at timestamp, regardless of outcome.

{% hint style="info" %}
**What counts as success:** An attempt is considered successful if the receiving endpoint responds with any 2xx status code within the configured timeout window. Any other response — including 3xx redirects, 4xx or 5xx errors, timeouts, or connection failures — counts as a failed attempt and triggers a retry, subject to the endpoint's retry policy.
{% endhint %}
