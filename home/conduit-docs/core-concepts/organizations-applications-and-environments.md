---
icon: sitemap
---

# Organizations, Applications, and Environments

**Conduit's account structure has three levels:**

1. **Organization:** The top-level account. An Organization contains one or more Applications and all of your team members. Billing is configured at the Organization level.
2. **Application:** A container representing one of your products or integration surfaces. Most customers start with a single Application; larger customers may create separate Applications per product line (for example, a company with both a billing product and a logistics product might run two Applications, each with its own endpoints and event types).
3. **Environment:** Every Application has exactly two environments: Sandbox and Live. Environments are fully isolated from each other, separate data, separate API keys, separate everything. There is no staging or custom environment tier; this is a deliberate simplicity constraint that keeps environment-related configuration predictable across the entire product.

The hierarchy is strict and one-directional: an Organization has many Applications; an Application has exactly two Environments; everything else (Endpoints, Events, API Keys) lives inside one Environment of one Application.

{% hint style="info" %}
Related reading: [Dashboard & Admin Guide](../guides/dashboard-and-admin-guide.md#managing-applications-and-environments) → Managing Applications and Environments; [Getting Started → Sandbox vs. Live.](../getting-started/sandbox-vs.-live-promoting-your-integration.md)
{% endhint %}
