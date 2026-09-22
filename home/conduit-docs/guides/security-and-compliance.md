---
icon: expeditedssl
---

# Security & Compliance

### Security Overview

Conduit's security model rests on four pillars, each covered in depth later in this section:

1. Encryption — in transit and at rest, everywhere data is handled.
2. Payload authenticity — every delivery is cryptographically signed so receiving systems can verify it originated from Conduit and wasn't altered.
3. Access control — API keys and Dashboard sessions are scoped and role-restricted, with no ambient trust between Applications or environments.
4. Data minimization and retention — payload data is retained only as long as necessary for operational and audit purposes, with a defined, finite retention window.

At a glance:

| **Property**             | **Detail**                                                          |
| ------------------------ | ------------------------------------------------------------------- |
| Encryption in transit    | TLS 1.2+                                                            |
| Encryption at rest       | AES-256                                                             |
| Payload signing          | HMAC-SHA256, per-endpoint secret                                    |
| Authentication           | API keys (Ingest/Admin API); email/password or SAML SSO (Dashboard) |
| Compliance certification | SOC 2 Type II                                                       |
| Data residency           | US (us-east-1) and EU (eu-west-1) regions available                 |
| Payload retention        | 30 days default, up to 90 days on Enterprise                        |

This overview is intentionally a summary. Each property is expanded with implementation detail in the sections that follow — treat this table as a starting checklist for a security review, not the review itself.

### Payload Signing and Verification: Deep Dive

The conceptual model is introduced in Core Concepts → Payload Signing. This section covers the mechanism in full, at the level of detail required for a security review or an independent implementation audit.

**Signature computation**

For every delivery attempt, Conduit computes:

```
signed_content = "{timestamp}.{raw_payload_body}"
signature = HMAC-SHA256(key=signing_secret, message=signed_content)
```

The resulting hex-encoded signature is sent in the Conduit-Signature header; the timestamp used is sent separately in Conduit-Timestamp (see API Reference → Webhooks Reference).

**Why the raw body, not a re-serialized version**

The signature is computed over the exact bytes Conduit transmits — not a parsed-and-re-serialized representation. This is why Guides → Verifying webhook signatures emphasizes reading the raw request body before any framework middleware parses it: JSON re-serialization can silently reorder keys or alter whitespace, producing a byte sequence that no longer matches the original signature even though the _data_ is identical. This is a common source of false-negative verification failures and is cataloged in Troubleshooting → Signature verification failures.

**Timestamp inclusion and replay-attack prevention**

Including the timestamp in the signed content, combined with a tolerance window enforced on the _receiving_ end (recommended: 300 seconds, as shown in Guides → Verifying webhook signatures), prevents an intercepted, validly-signed payload from being replayed by an attacker at a later time. The signature alone would remain valid indefinitely without the timestamp check — the tolerance window is a receiving-side control, not something Conduit enforces on your behalf, and should be implemented explicitly in your verification code.

**Signing scheme scope**

Conduit supports HMAC-SHA256 exclusively. There is no asymmetric signing option (e.g., RSA) in the current version. This is a deliberate scope decision — see Section 22.5 of the product foundation — that keeps verification code identical across every supported language and consistent with the pattern established by comparable webhook systems (Stripe, GitHub).

For a security review: HMAC-SHA256 with a shared secret means both Conduit and your endpoint hold the same secret material. This is the correct threat model for verifying _authenticity and integrity_ of a delivery from a trusted platform (Conduit) to your infrastructure — it is not intended to provide non-repudiation between mutually distrusting parties, which would require asymmetric signing.

### Signing Secret Lifecycle and Rotation

Generation: A unique signing secret is generated at endpoint creation and returned exactly once, in the creation response (API Reference → §5.5.2). Conduit does not store secrets in a form that can be redisplayed — only rotation, not retrieval, is available if a secret is lost.

Rotation model: Rotation is additive, not a hard cutover. When a new secret is generated:

* The old secret remains valid for a configurable grace period.
* Deliveries during the grace period are signed with the new secret; your endpoint's verification code should accept either secret during this window (see the rollout procedure in Guides → Rotating signing secrets safely, and FIG-DIAG-03 for the grace-period timeline).
* After the grace period expires, the old secret is invalidated automatically and permanently.

Why this matters for a security review: The grace-period model exists specifically to avoid a forced choice between security hygiene (regular rotation) and delivery continuity (zero dropped deliveries during rotation). A rotation policy that required an instantaneous cutover would create pressure to rotate infrequently; the overlapping-validity model removes that tradeoff.

Recommended rotation cadence: Conduit does not enforce a rotation schedule. Organizations with formal secret-rotation policies (common under SOC 2 and similar frameworks) should implement rotation on their own defined cadence using the procedure in Guides → Rotating signing secrets safely; the Admin API supports full automation of this via POST /v1/endpoints/{id}/rotate-secret.

### Authentication and Authorization Model

For your systems calling Conduit's APIs:

API keys are the sole authentication mechanism for the Ingest and Admin APIs — there is no OAuth flow (see API Reference → Authentication). Each key is immutably scoped to one Application and one Environment at creation, enforced server-side on every request, not merely in client-side tooling. A request made with a Sandbox key against a Live resource fails with 403 / environment\_mismatch regardless of the caller's intent — see API Reference → Errors.

For your endpoints, verifying deliveries from Conduit:

Each delivery carries an HMAC-SHA256 signature computed with the receiving endpoint's own signing secret, described fully in §8.2 above.

For human users of the Dashboard:

Email/password authentication, or SAML-based SSO on the Enterprise plan (setup procedure in Dashboard & Admin Guide → SSO setup). Dashboard sessions use JWTs, and every Dashboard and Admin API action is subject to the same role-based access control model — a single, consistent permissions system rather than separate rules for UI vs. API access (see Core Concepts and the full permissions table in Dashboard & Admin Guide → Team management and role assignment).

### Data Retention and Deletion Policy

Delivery Log payload retention: Full request and response bodies logged for each Delivery Attempt are retained for 30 days by default, configurable up to 90 days on the Enterprise plan. After the retention window, payload bodies are purged; metadata (status, response codes, timestamps) may be retained longer for aggregate reporting purposes, without the associated payload content.

Event payload retention: Events themselves follow the same retention model as their associated Delivery Attempts, since an Event's payload and its Delivery history are logically linked for audit purposes.

Data deletion on Application deletion: Deleting an Application (Dashboard & Admin Guide → Managing Applications and Environments) permanently removes all associated Endpoints, Events, Deliveries, and API Keys immediately — this is not subject to the standard retention window, since it is an explicit, confirmed deletion action rather than a passive expiry.

Why a finite, stated retention window: A fixed, published retention period gives your own data retention and compliance documentation something concrete to reference, rather than an open-ended "logs are kept" commitment with no defined bound — this is a deliberate design decision (see Section 22.7 of the product foundation) intended to make Conduit's data handling auditable by design.

### Compliance Certifications

**SOC 2 Type II:** Conduit is SOC 2 Type II compliant. Reports are available under NDA through your Lattice Systems account representative or via the Trust Center (§8.10).

**HIPAA / BAAs:** Business Associate Agreements are available for Enterprise plan customers handling healthcare data, consistent with the compliance requirements common among Conduit's target customer profile (see Persona 4 — Alex Rutherford in the product foundation, and the "compliance requirements (SOC 2, HIPAA)" criterion in the ideal customer profile).

**Requesting compliance documentation:** SOC 2 reports, sub-processor lists, and BAA templates are requested through the Trust Center or your account representative — these are not self-service downloads from the Dashboard, consistent with standard practice for documents requiring an NDA or signature.

### Network Security

Conduit delivers webhooks from a published, static set of egress IP ranges, allowing customers to allowlist Conduit at their own firewall rather than opening inbound access broadly.

**Current egress IP ranges:**

203.0.113.0/24    (us-east-1)

198.51.100.0/24   (eu-west-1)

Egress ranges are subject to change with advance notice. Subscribe to the Changelog or the status page for update announcements before hardcoding these ranges into firewall rules without a review process.

All inbound API traffic to Conduit (Ingest API, Admin API, Dashboard) is served exclusively over TLS 1.2+; unencrypted HTTP requests are rejected outright, not redirected.

Outbound delivery TLS requirements: Endpoint URLs must be HTTPS — Conduit rejects HTTP-only endpoint URLs at registration time (422 / invalid\_url, see API Reference → Endpoints), since payload confidentiality and integrity in transit is a baseline requirement, not an optional hardening step.

### Deployment Models and Data Residency

Conduit is offered exclusively as a cloud service — there is no self-hosted or on-premises deployment option, and documentation throughout this suite should not be interpreted as implying otherwise. This is a deliberate product scoping decision (Section 22.1 of the product foundation), not an oversight: it keeps the entire documentation suite focused on integration rather than installation and operations content.

Two cloud deployment models are available:

| **Model**                       | **Description**                                                                                          | **Availability** |
| ------------------------------- | -------------------------------------------------------------------------------------------------------- | ---------------- |
| Multi-tenant managed cloud      | The default and primary model, at conduit.dev                                                            | All plans        |
| Single-tenant / dedicated cloud | Deployed into a dedicated AWS account per customer, for strict data residency or compliance requirements | Enterprise only  |

**Regions:**

* us-east-1 — primary region
* eu-west-1 — secondary region, for customers with EU data residency requirements

Region is selected at Application creation and is not changeable afterward in the current version (see Guides → Multi-region and data residency considerations). All stateful components (the Event Bus and Delivery Log Store) are deployed multi-AZ within the selected region.

For a security review evaluating self-hosted requirements: If your organization's policy requires on-premises deployment, Conduit's dedicated-cloud option (above) is the closest available model — an isolated AWS account per customer, though still cloud-hosted rather than customer-infrastructure-hosted. Confirm this fits your requirements before proceeding with adoption.

### Responsible Disclosure and Security Contact

Lattice Systems maintains a responsible disclosure program for security researchers.

Reporting a vulnerability: security@latticesystems.example — PGP key available at [latticesystems.example/security.txt](https://test.com).

{% hint style="warning" %}
Disclosure policy: Reports are acknowledged within 2 business days. Lattice Systems requests a minimum 90-day disclosure window before public disclosure, to allow time for remediation and coordinated release.
{% endhint %}

**Scope:** The disclosure program covers \*.conduit.dev and the Ingest/Admin APIs. Social engineering, physical security, and denial-of-service testing are out of scope — see the full policy at the link above for the complete rules of engagement.

### Sub-Processor List

A current list of third-party sub-processors used in delivering Conduit — cloud infrastructure providers, monitoring tooling, and similar — is maintained on the Trust Center rather than in this documentation directly, since sub-processor relationships change independently of product documentation release cycles.

Notification of sub-processor changes: Enterprise customers are notified in advance of any new sub-processor addition, with an objection window consistent with standard data processing agreement terms — see your Organization's DPA for the exact notice period.

***

#### Next Steps

* Implementing signature verification code → Guides → Verifying webhook signatures
* Diagnosing a signature verification failure → Troubleshooting → Signature verification failures
* Rotating a signing secret operationally → Guides → Rotating signing secrets safely
* Requesting compliance documentation for procurement → Trust Center (linked above) or your Lattice Systems account representative
