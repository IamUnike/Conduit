---
icon: file-circle-check
---

# Payload Signing

Every delivery Conduit sends is signed, so your endpoint can verify that a payload genuinely originated from Conduit and was not altered in transit or forged by a third party.

**At a conceptual level:**

1. Each endpoint has a unique signing secret, generated when the endpoint is created and shown only once.
2. When Conduit delivers a payload, it computes an HMAC-SHA256 signature over the delivery timestamp and the raw request body, using that secret.
3. The signature is sent in a Conduit-Signature request header alongside the payload.
4. Your endpoint independently recomputes the same signature using its copy of the signing secret and compares it to the header value. A match confirms authenticity; a mismatch means the payload should be rejected.

This is the same general pattern used by Stripe- and GitHub-style webhook verification, and Conduit supports HMAC-SHA256 exclusively. There is no asymmetric signing option in the current version, which keeps verification code identical across every language and integration.

{% hint style="info" %}
Full verification code samples, secret rotation procedures, and replay-attack prevention (timestamp tolerance windows) are covered in depth in [Security & Compliance → Payload signing and verification ](../guides/security-and-compliance.md#payload-signing-and-verification-deep-dive)deep dive. A working verification example for your first integration is in [Getting Started → Step 5.](../getting-started/getting-started.md)
{% endhint %}
