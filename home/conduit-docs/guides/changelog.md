---
icon: swap
---

# Changelog

### Product Changelog

Tracks Dashboard and platform-level features — additions visible primarily through the Dashboard or account configuration, as opposed to API schema changes (§10.2, below).

**Format and conventions:**

* Entries are grouped by release date, most recent first.
* Each entry is tagged Added, Improved, or Fixed.
* Entries affecting only a specific plan tier are labeled accordingly (e.g., _Enterprise only_).
* This log does not include API-level changes — see §10.2 for those.

**Example entries:**

`## 2026-07-01`

`### Added`

_- Bulk replay from the Dead Letter Queue screen — select multiple_

&#x20; _entries and replay in a single action (see Dashboard & Admin_

&#x20; _Guide → Replaying deliveries from the Dashboard)_

_- Delivery health overview panel on the Endpoints detail page_

&#x20; _(success rate, average response time, DLQ count)_&#x20;

`### Improved`

_- Delivery Log search now supports filtering by date range in_

&#x20; _addition to event ID, endpoint, and status_

`## 2026-06-15`&#x20;

`### Added`

_- SAML SSO for Dashboard authentication (Enterprise only) — see_

&#x20; _Dashboard & Admin Guide → SSO Setup_

`### Fixed`

_- Endpoint activity history now correctly attributes status changes_

&#x20; _made via the Admin API, not only Dashboard-initiated changes_

`## 2026-05-20`&#x20;

`### Added`

_- Event filtering and transformation rules — endpoints can now_

&#x20; _receive a subset of a subscribed event type based on payload_

&#x20; _field conditions (see Guides → Setting up event filtering and_

&#x20; _transformation)_

### API Changelog

Tracks schema-level changes to the Ingest and Admin APIs specifically. This is the changelog referenced from API Reference → §5.7 — reproduced here as part of the consolidated Changelog section for readers who don't arrive via the API Reference directly.

**Format and conventions:**

* Every entry is tagged Added (non-breaking) or Changed (breaking).
* Breaking changes are never introduced into an existing API version (v1) — see API Reference → Base URL, Versioning, and Conventions. A Changed entry always corresponds to a new major version and includes a migration guide link.
* Non-breaking additions (new optional fields, new endpoints, new error codes) ship into the current version directly.

**Example entries:**

`## 2026-07-01`

`### Added`

_- \`filters\` field on Endpoint resource (non-breaking) — see API_

&#x20; _Reference → Endpoints_

_- \`replay\_in\_progress\` error code (409) for concurrent replay_

&#x20; _attempts on the same delivery_&#x20;

`## 2026-05-15`

`### Changed — v2 only, v1 unaffected`

_- GET /v1/deliveries pagination model will switch from offset-based_

&#x20; _to cursor-based in v2. v1 continues to support offset pagination_

&#x20; _indefinitely. Migration guide: \[link]_

`## 2026-04-10`

`### Added`

_- POST /v1/endpoints/{id}/rotate-secret endpoint (non-breaking) —_

&#x20; _see Guides → Rotating signing secrets safely_

### Deprecation Notices and Migration Timelines

Consolidates every active deprecation across the API and SDKs into a single forward-looking view, so a reader can confirm whether anything they currently depend on has an end-of-life date.

**Format:**

| **Deprecated item**                                       | **Deprecated on** | **Removal date**                              | **Replacement**         | **Migration guide**                                                            |
| --------------------------------------------------------- | ----------------- | --------------------------------------------- | ----------------------- | ------------------------------------------------------------------------------ |
| _(example)_ Offset-based pagination on GET /v1/deliveries | 2025-05-15        | Not before v2 general availability + 6 months | Cursor-based pagination | [Migration guide](https://claude.ai/chat/7ce846f0-fe72-4bbc-8129-11314198df5f) |

**Deprecation policy:**

* No breaking change is introduced without a published replacement available first.
* Deprecated functionality remains supported for a minimum of 6 months after its replacement ships, or until the next major API version, whichever is longer.
* Deprecation notices are also surfaced as Deprecation response headers on affected API calls, so automated integrations can detect upcoming changes without manually monitoring this page.

{% hint style="info" %}
Subscribe to the Changelog RSS feed or the status page to be notified of new deprecation entries as they're published, rather than checking this page manually on a recurring basis.
{% endhint %}

***

#### Next Steps

* Confirming whether a specific field or endpoint is deprecated → §10.3 above
* Full schema for a resource mentioned in a changelog entry → API Reference
* Understanding why a design decision was made a certain way → Core Concepts, or the relevant Security & Compliance page for security-related decisions
