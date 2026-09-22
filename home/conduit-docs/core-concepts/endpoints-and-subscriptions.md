---
icon: arrows-to-circle
---

# Endpoints and Subscriptions

An Endpoint is a URL you register with Conduit to receive deliveries. Each endpoint has:

* A destination url
* One or more subscribed event\_types
* A signing\_secret, used to authenticate deliveries to that endpoint
* A retry\_policy&#x20;
* A status — active or paused

A Subscription is the association between an endpoint and the event types it should receive. An endpoint with no subscriptions will not receive any deliveries, even if events are actively flowing through the Application — this is a common source of "why didn't my webhook arrive" confusion and is covered in [Troubleshooting → My webhook didn't arrive.](../guides/troubleshooting-and-faq.md#my-webhook-didnt-arrive-diagnostic-flowchart)

Endpoints can be paused without being deleted. A paused endpoint stops receiving new deliveries but retains its configuration, subscriptions, and history — useful when a receiving system is undergoing maintenance and you want to avoid accumulating failed delivery attempts.

{% hint style="info" %}
**Multiple endpoints per event type:** More than one endpoint can subscribe to the same event type. When this happens, Conduit creates one independent Delivery per subscribed endpoint — a failure on one endpoint has no effect on delivery to the others.
{% endhint %}
