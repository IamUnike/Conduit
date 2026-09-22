---
icon: screen-users
---

# Team & Account Guides

### **Inviting Team Members and Assigning Roles**

Roles are scoped per Application, a team member can hold different roles on different Applications within the same Organization. See [Core Concepts → Organizations, Applications, and Environments ](../core-concepts/organizations-applications-and-environments.md)and the full permissions table in [Dashboard & Admin Guide → Team management and role assignment.](dashboard-and-admin-guide.md#team-management-and-role-assignment)

**Inviting a member via the Dashboard:**

1. Go to Team → Invite Member.
2. Enter their email and select a role (Owner, Admin, Developer, or Viewer) for the current Application.
3. Repeat per Application if they need access to more than one.

### **Managing Multiple Applications Under One Organization**

Recommended when your Organization has genuinely separate products or integration surfaces **(see** [Core Concepts → Organizations, Applications, and Environments](../core-concepts/organizations-applications-and-environments.md) for when to split vs. keep a single Application).

Switching between Applications is available from the Dashboard's Application selector, and the Admin API scopes every request to a single Application implicitly via the API key used — there is no cross-Application query available in a single API call by design, reinforcing Application-level data isolation.

### Billing and Usage Monitoring

Conduit bills on delivered events per month, at the Organization level, aggregated across all Applications and both environments (Sandbox usage is not billed — see [Getting Started → Sandbox vs. Live](../getting-started/sandbox-vs.-live-promoting-your-integration.md)**).**

**Checking current usage:**

Go to Organization Settings → Billing in the Dashboard for a real-time usage summary against your plan's included volume, and historical invoices.

Plan tiers (Free, Business, Enterprise) and their included volumes, rate limits, and feature gating are detailed in the [API Reference → Rate limits](api-reference.md#rate-limits-and-pagination) and a dedicated Pricing page. Feature availability by tier (SSO, dedicated cloud, extended retention) is called out throughout this documentation wherever relevant.
