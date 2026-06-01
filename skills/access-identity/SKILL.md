---
name: "access-identity"
description: "Configure identity providers (IdP) and SCIM provisioning for Cloudflare Access. Covers IdP setup, SCIM sync, group mapping, multi-IdP strategies, and identity-based policy selectors."
domain: "access"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/identity/"
---

# Access Identity & IdP Configuration

## Persona

You are a senior Cloudflare One architect specializing in identity integration. You design IdP configurations that enable granular access policies while maintaining a smooth authentication experience for end users.

## When to Use

- Customer is connecting an identity provider (Okta, Azure AD, OneLogin, etc.) to CF1
- Customer needs SCIM provisioning for group-based policies
- Customer has multiple IdPs and needs a multi-IdP strategy
- Customer asks about identity selectors, Access groups, or SCIM sync

## When NOT to Use

- Customer needs to create Access applications (not configure identity) → use `access-app-setup`
- Customer needs Gateway identity selectors without Access → still use this skill (SCIM feeds both)

## Assess

1. **Identify current IdP(s)**:
   → execute: GET `/accounts/${accountId}/access/identity_providers` — list all configured IdPs
   → execute: GET `/accounts/${accountId}/access/identity_providers/${idpId}` — check SCIM config status on each

2. **Determine SCIM provisioning status**:
   → execute: GET `/accounts/${accountId}/access/identity_providers/${idpId}/scim/groups` — check if groups are syncing
   → execute: GET `/accounts/${accountId}/access/identity_providers/${idpId}/scim/users` — check if users are syncing
   - If SCIM is not configured, only email, email domain, and service token selectors are available for policies.

3. **Inventory existing Access Groups**:
   → execute: GET `/accounts/${accountId}/access/groups` — list reusable groups already created
   - Groups simplify policy management when the same conditions apply across apps.

4. **Check existing identity usage in policies**:
   → execute: GET `/accounts/${accountId}/access/policies` — review which identity selectors are in use across reusable policies

5. **Key questions to gather**:
   - Which IdP(s) does the customer use? (Okta, Azure AD, Google Workspace, OneLogin, SAML generic, OIDC generic)
   - Is SCIM supported by their IdP plan?
   - What is the group structure? (flat vs. nested, naming conventions)
   - Are there existing SSO federation setups to preserve?
   - Is there a multi-IdP requirement (e.g., employees via Okta, contractors via Azure AD)?

## Design

### Identity Selector Hierarchy

| Selector | Requires SCIM? | Use Case |
|---|---|---|
| `email` | No | Individual user access, testing |
| `email_domain` | No | Allow entire domain (e.g., `@company.com`) |
| `saml_group_id` / `group` | **Yes** | Group-based access (recommended for production) |
| `service_token` | No | Machine-to-machine / automation |
| `device_posture` | No (separate) | Device health requirements |
| `ip_list` | No | IP-based restrictions (Access only supports IP lists, not domain/URL) |
| `country` | No | Geographic restrictions |
| `certificate` | No | mTLS client certificate |

### SCIM Provisioning Design

- SCIM provisioning is **required** for group-based policy selectors (`identity.groups.name` in Gateway, `saml_group_id` in Access).
- Without SCIM, you can still gate on email, email domain, and service token — but group-based policies are the production standard.
- SCIM syncs groups and users automatically from IdP → Cloudflare. Changes propagate within minutes depending on IdP.

### Multi-IdP Strategy

- Multiple IdPs are supported simultaneously. Each is configured independently.
- When `auto_redirect_to_identity` is enabled on an app with multiple IdPs, users see an IdP selection screen.
- For single-IdP setups, enable `auto_redirect_to_identity` for seamless login.
- Common pattern: primary IdP for employees (Okta), secondary for contractors (Azure AD B2B or SAML generic).

### Access Groups Design

- **Access Groups** are CF1 constructs — reusable sets of include/exclude/require rules.
- Access Groups are NOT the same as IdP groups. An Access Group can reference IdP groups (via SCIM), email domains, or any other selector.
- Use Access Groups to simplify policy attachment across many apps (e.g., "Engineering Team" group used in 50+ app policies).

## Implement

### With MCP Tools (Code Mode)

#### A. Review Existing IdP Configuration

1. **List all identity providers**:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/access/identity_providers` })
   - Note the `id`, `type`, and `name` of each IdP.

2. **Check SCIM status on each IdP**:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/access/identity_providers/${idpId}` })
   - Check `scim_config` in the response. If absent or disabled, SCIM needs to be configured via Dashboard.

3. **List synced SCIM groups**:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/access/identity_providers/${idpId}/scim/groups` })
   - Record group IDs for use in policy selectors.

4. **Verify synced users**:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/access/identity_providers/${idpId}/scim/users` })
   - Confirm expected users appear. Missing users = SCIM provisioning incomplete.

#### B. Create Access Groups

1. **Create a reusable Access Group** for a team:
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/access/groups`,
       body: { name: "Engineering Team", include: [{ saml: { attribute_name: "groups", attribute_value: "Engineering" } }] }
     })

2. **Create a group with multiple selectors** (OR logic within include):
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/access/groups`,
       body: { name: "All Employees", include: [{ email_domain: { domain: "company.com" } }, { email_domain: { domain: "subsidiary.com" } }] }
     })

3. **Create a group with require rules** (AND logic):
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/access/groups`,
       body: { name: "Secure Engineering", include: [{ saml: { attribute_name: "groups", attribute_value: "Engineering" } }], require: [{ device_posture: { integration_uid: "<posture-rule-id>" } }] }
     })

#### C. Create Reusable Policies Referencing Groups

1. **Create a reusable Allow policy**:
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/access/policies`,
       body: { name: "Allow Engineering", decision: "allow", include: [{ group: { id: "<access-group-id>" } }] }
     })

2. This policy can then be attached to multiple Access applications.

### Without MCP Tools (Dashboard/API)

1. Navigate to **Zero Trust → Settings → Authentication** to add/configure identity providers.
2. For SCIM: In the IdP settings, enable SCIM provisioning and copy the SCIM endpoint + token to your IdP admin console.
3. Navigate to **Zero Trust → Access → Access Groups** to create reusable groups.
4. Navigate to **Zero Trust → Access → Policies** to create reusable policies.
5. When creating an Access application, select which IdPs are allowed and attach reusable policies.

## Validate

1. **Verify IdP configuration**:
   → execute: GET `/accounts/${accountId}/access/identity_providers` — confirm all expected IdPs appear
   → execute: GET `/accounts/${accountId}/access/identity_providers/${idpId}` — confirm SCIM config is active

2. **Verify SCIM sync**:
   → execute: GET `/accounts/${accountId}/access/identity_providers/${idpId}/scim/groups` — confirm expected groups appear
   → execute: GET `/accounts/${accountId}/access/identity_providers/${idpId}/scim/users` — confirm expected users appear
   - If groups/users are missing, check SCIM provisioning in the IdP admin console.

3. **Verify Access Groups**:
   → execute: GET `/accounts/${accountId}/access/groups` — confirm groups exist with correct selectors
   → execute: GET `/accounts/${accountId}/access/groups/${groupId}` — inspect include/exclude/require rules

4. **Test authentication flow**:
   - Create a test Access application and attach a policy referencing an Access Group.
   - Authenticate as a user in the group → should succeed.
   - Authenticate as a user NOT in the group → should be denied.

5. **Check user identity resolution**:
   → execute: GET `/accounts/${accountId}/access/users` — confirm users who have authenticated appear
   → execute: GET `/accounts/${accountId}/access/users/${userId}/last_seen_identity` — verify group membership and device posture attributes are present in the identity

6. **Review access logs**:
   → execute: GET `/accounts/${accountId}/access/logs/access_requests` — check for successful and denied authentication events

## Operate

1. **SCIM sync issues** (users/groups missing or stale):
   → GET `/accounts/${accountId}/access/identity_providers/${idpId}/scim/groups` / `scim/users` to check current state
   - If stale, the issue is in the IdP's SCIM provisioning, not CF1. Check IdP admin console for sync errors.
   - SCIM sync frequency varies by IdP: Okta ~40min, Azure AD ~40min, Google Workspace ~24h.

2. **User can't authenticate** (identity-related):
   → GET `/accounts/${accountId}/access/users` — check if user exists in CF1
   → GET `/accounts/${accountId}/access/users/${userId}/failed_logins` — review failure reasons
   → GET `/accounts/${accountId}/access/users/${userId}/last_seen_identity` — check if group claims are present
   → GET `/accounts/${accountId}/access/logs/access_requests` — search for denied events with the user's email

3. **Add new IdP**:
   - Configure via Dashboard (Zero Trust → Settings → Authentication → Add new).
   - Enable SCIM if supported. Verify with GET `/accounts/${accountId}/access/identity_providers/${idpId}/scim/groups` after provisioning.

4. **Update Access Group membership**:
   → PUT `/accounts/${accountId}/access/groups/${groupId}` with updated include/exclude/require rules
   - Changes take effect immediately for new authentication events. Existing sessions are not revoked.

5. **Rotate/audit service tokens**:
   - Service tokens are managed by the `access-app-setup` skill, but identity-related audit includes verifying which apps use `service_auth` policies.

## Gotchas

1. Group names are **CASE-SENSITIVE** between IdP and CF1 — `"Engineering"` ≠ `"engineering"`. SCIM syncs the exact case from the IdP.
2. SCIM sync frequency varies by IdP — changes may take **minutes to hours** to propagate. Okta/Azure ~40min push, Google Workspace up to 24h.
3. **Access Groups ≠ IdP Groups** — Access Groups are CF1 constructs that can reference IdP attributes. An Access Group named "Engineering" doesn't automatically map to an IdP group named "Engineering" — you must explicitly include the IdP group selector.
4. Gateway identity selectors (`identity.groups.name`) use **SCIM groups**, NOT Access Groups. Access policies use Access Groups (which may reference SCIM groups internally).
5. **IP lists only** in Access policy selectors — domain lists and URL lists are Gateway-only.
6. When SCIM is disabled or broken, any policy using group selectors will **fail closed** (deny access), because the group claim won't be present in the user's identity.
7. `access_user_last_seen_identity` shows the identity from the **last authentication**, not real-time. If a user's group membership changed, the stale identity persists until re-authentication.

## Related Skills

- **access-app-setup**: Create Access applications that use identity-based policies
- **access-saas-federation**: SaaS SSO federation (depends on IdP setup)
- **device-posture-setup**: Device posture rules often combined with identity in policies
- **gateway-policy-design**: Gateway policies use SCIM groups for identity-based filtering
