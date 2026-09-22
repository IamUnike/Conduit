---
description: Roles, permissions, and how access flows from workspace to project.
icon: lock
---

# Permissions

An Event is a single, discrete occurrence that you publish into Conduit for delivery — an invoice being paid, a user being created, a shipment status changing. Each event has:

| **Field**   | **Description**                           |
| ----------- | ----------------------------------------- |
| id          | Unique identifier assigned by Conduit     |
| event\_type | The category/schema this event belongs to |
| payload     | The event's data, as JSON                 |
| created\_at | Timestamp of ingestion                    |
| environment | Sandbox or Live                           |

An Event Type is the category an event belongs to — for example, invoice.paid, user.created, or subscription.cancelled. Event types are not pre-registered with Conduit; you define them implicitly the first time you publish an event of that type, and endpoints subscribe to event types by name.

> Naming convention: Conduit does not enforce an event type naming scheme, but the convention used throughout this documentation — and recommended for your own integration — is resource.action in lowercase with dot separators (e.g., invoice.paid, not InvoicePaid or invoice\_paid). Consistent naming makes subscription management significantly easier as your event catalog grows.

Once published, an event is immutable — it cannot be edited or deleted. If a mistake was published, publish a correcting event rather than attempting to alter the original; this preserves the delivery log's accuracy as an audit trail.
