---
icon: compass
---

# Sandbox vs. Live: Promoting Your Integration

Every Application has exactly two environments: **Sandbox and Live**. This is a deliberate design constraint. Conduit does not support staging or custom environments, which keeps environment-related configuration simple and predictable.

|                    | **Sandbox**              | **Live**                    |
| ------------------ | ------------------------ | --------------------------- |
| **Purpose**        | Development and testing  | Production traffic          |
| **API key prefix** | sk\_sandbox\_...         | sk\_live\_...               |
| **Data isolation** | Fully separate from Live | Fully separate from Sandbox |
| **Billing**        | Not billed               | Billed per delivered event  |

A Sandbox API key cannot read or write Live data, and vice versa — this isolation is enforced at the API level, not just in the Dashboard UI, so a misconfigured key fails safely rather than silently crossing environments.

**Promoting your integration**

There is no automatic "promote" action — Sandbox and Live are independently configured. To go live:

1. Switch to the Live environment in the Dashboard (top-left environment selector).
2. Recreate your endpoint registrations and event type subscriptions (or use the Admin API to script this — see [Guides → Setting up multiple environments in CI/CD](../guides/integration-guides.md)).
3. Generate a Live API key and update your production configuration.
4. Update your signature verification code to use the Live endpoint's signing secret — it is different from the Sandbox secret.

{% hint style="warning" icon="triangle-exclamation" %}
Common mistake: Using a Sandbox signing secret to verify Live deliveries (or vice versa) is the single most common integration error. See Troubleshooting → Signature verification failures.
{% endhint %}
