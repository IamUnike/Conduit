---
description: >-
  This is the single canonical source for every defined term used across the
  Conduit documentation suite.
icon: glasses-round
---

# Glossary

| **Term**                | **Definition**                                                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Admin API               | The API used to manage Applications, Endpoints, API Keys, and Team membership — as distinct from the Ingest API, which is used only to publish events. |
| API Key                 | A credential used to authenticate API requests, scoped to exactly one Application and one Environment.                                                 |
| Application             | A container representing one of a customer's products or integration surfaces in Conduit; the top-level unit of configuration beneath an Organization. |
| Dead Letter Queue (DLQ) | Storage for Deliveries that have exhausted all configured retry attempts without a successful response.                                                |
| Delivery                | One Event's delivery lifecycle to one specific Endpoint.                                                                                               |
| Delivery Attempt        | A single HTTP request made as part of a Delivery's retry sequence.                                                                                     |
| Endpoint                | A URL registered to receive deliveries for one or more Event Types.                                                                                    |
| Environment             | Sandbox or Live — isolated data, API keys, and configuration within an Application. Every Application has exactly two.                                 |
| Event                   | A discrete occurrence published into Conduit for delivery. Immutable once created.                                                                     |
| Event Type              | The category or schema an Event belongs to (e.g., invoice.paid), conventionally written as resource.action.                                            |
| Ingest API              | The API customers use to publish Events into Conduit.                                                                                                  |
| Organization            | The top-level Conduit account, containing one or more Applications and all team members; billing is configured at this level.                          |
| Replay                  | Manually re-triggering delivery of a past Event or Delivery, outside the automatic retry sequence.                                                     |
| Retry Policy            | The configured backoff schedule — attempt count and delay curve — applied to a failed Delivery.                                                        |
| Signing Secret          | The per-Endpoint secret used to compute the HMAC-SHA256 signature on outbound payloads.                                                                |
| Subscription            | The association between an Endpoint and the Event Types it should receive.                                                                             |
