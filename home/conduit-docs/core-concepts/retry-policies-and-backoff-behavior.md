---
icon: rotate
---

# Retry Policies and Backoff Behavior

A Retry Policy is the configured schedule Conduit follows when a delivery attempt fails — how many times to retry, and how long to wait between attempts. Retry policies are configured per endpoint, so different endpoints on the same Application can have different tolerances for retry aggressiveness.

Conduit uses exponential backoff: the wait time between attempts increases with each successive failure, rather than retrying at a fixed interval. This avoids two failure modes common in hand-built retry systems:

* Too aggressive — hammering a downed endpoint with rapid retries, which can worsen an outage on the receiving side.
* Too passive — fixed, generous delays that leave a transient failure (e.g., a 10-second deploy window) unresolved for far longer than necessary.

The default retry policy retries a failed delivery attempt a configurable number of times, with the delay between attempts roughly doubling after each failure, up to a maximum delay ceiling. Once the configured attempt count is exhausted, the delivery is marked exhausted and moves to the Dead Letter Queue.

{% hint style="info" %}
For guidance on configuring retry policy parameters for your own endpoints, see [Guides → Configuring retry policies.](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/guides)
{% endhint %}
