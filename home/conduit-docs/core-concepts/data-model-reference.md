---
icon: container-storage
---

# Data Model Reference

The table below consolidates every entity introduced in Core Concepts into a single reference. Use it as a map back to the relevant subsection whenever a relationship between entities is unclear.

| **Entity**       | **Key fields**                                                                       |
| ---------------- | ------------------------------------------------------------------------------------ |
| Application      | id, name, created\_at                                                                |
| Endpoint         | id, url, event\_types\[], signing\_secret, retry\_policy, status                     |
| Event            | id, event\_type, payload, created\_at, environment                                   |
| Delivery         | id, event\_id, endpoint\_id, status, attempts\[]                                     |
| Delivery Attempt | id, delivery\_id, attempt\_number, response\_code, response\_time\_ms, attempted\_at |
| API Key          | id, prefix, environment, created\_at, last\_used\_at                                 |

***

### Next Steps

With the conceptual model in place, choose your next document based on what you're trying to do:

* Ready to build → [Guides ](https://app.gitbook.com/s/nhG8JL9P1uwPxfUoife2/guides)— task-oriented instructions for endpoint management, idempotency, retry configuration, and secret rotation.
* Need exact request/response schemas → [API Reference](../guides/api-reference.md).
* Preparing for a security review →[ Security & Compliance ](../guides/security-and-compliance.md)— full signing internals, retention policy, and compliance certifications.
* Supporting end users → [Dashboard & Admin Guide](../guides/dashboard-and-admin-guide.md)
