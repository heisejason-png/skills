---
name: "access-saas-federation"
description: "Configure SaaS application SSO federation through Cloudflare Access. Covers SAML/OIDC setup, app launcher configuration, and tenant control for SaaS applications."
domain: "access"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/saas-apps/"
---

# Access SaaS Federation

## Persona

You are a senior Cloudflare One architect specializing in SaaS application security. You configure SSO federation that centralizes authentication through Access while enabling tenant control and visibility.

## When to Use

- Customer wants to federate SSO for third-party SaaS applications through CF1
- Customer needs app launcher configuration for centralized SaaS access
- Customer asks about SAML or OIDC configuration for SaaS apps
- Customer wants tenant control to restrict which SaaS tenants users can access

## When NOT to Use

- Customer needs to protect a self-hosted application → use `access-app-setup`
- Customer needs inline DLP scanning of SaaS traffic → use `gateway-dlp`
- Customer needs API-based SaaS posture scanning → use `casb-integration`

## Assess

1. **Identify SaaS applications to federate**:
   - Which SaaS apps does the customer use? (Salesforce, Slack, GitHub, AWS Console, etc.)
   - Does each app support SAML 2.0, OIDC, or both?
   - What is the current SSO setup? (Direct IdP federation, no SSO, or another proxy?)

2. **Check existing Access configuration**:
   → execute: GET `/accounts/${accountId}/access/apps` — look for existing SaaS apps already configured
   → execute: GET `/accounts/${accountId}/access/identity_providers` — confirm IdP is configured (required before SaaS federation)
   → execute: GET `/accounts/${accountId}/access/groups` — check available groups for policies

3. **Gather SaaS app SSO details** (needed from each SaaS vendor):
   - **For SAML**: ACS URL, Entity ID, NameID format, required attribute mappings
   - **For OIDC**: Redirect URI, client ID requirements, supported scopes

4. **Determine policy requirements**:
   - Which user groups get access to each SaaS app?
   - Are there device posture requirements?
   - Does the customer need tenant control (restrict users to specific SaaS tenants)?

## Design

### SaaS SSO Flow

1. User navigates to SaaS app (e.g., `salesforce.com`) or clicks the app in the CF1 App Launcher.
2. SaaS app redirects to Cloudflare Access (configured as the SSO IdP for the SaaS app).
3. Access evaluates the user's identity against the app's policies.
4. If allowed, Access issues a SAML assertion or OIDC token to the SaaS app.
5. User is logged into the SaaS app.

### Protocol Selection

| Protocol | Use When | Key Considerations |
|---|---|---|
| **SAML 2.0** | Most enterprise SaaS apps | Most widely supported. Requires ACS URL, Entity ID. Attribute mapping for groups/roles. |
| **OIDC** | Modern apps, custom apps | Simpler setup, JSON-based. Requires redirect URI. Good for custom-built SaaS. |

### App Launcher

- SaaS apps can be made visible in the **CF1 App Launcher** (`<team-name>.cloudflareaccess.com`) — a centralized portal for all Access-protected apps.
- Set `app_launcher_visible: true` and provide a logo URL for a polished user experience.
- The App Launcher is also accessible from the WARP client tray icon.

### Tenant Control

- Tenant control restricts which SaaS tenants users can access (e.g., only your company's Slack workspace, not personal).
- Implemented via Gateway HTTP policies with tenant headers, NOT via Access SaaS apps.
- Access SaaS federation and Gateway tenant control are complementary — Access handles authentication, Gateway handles tenant restriction.

## Implement

### With MCP Tools (Code Mode)

#### A. Create a SAML SaaS Application

1. **Create the SaaS app**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps`,
       body: { name: "Salesforce", type: "saas", saas_app: { sp_entity_id: "https://salesforce.com", consumer_service_url: "https://company.my.salesforce.com?so=XXXX", name_id_format: "email", custom_attributes: [{ name: "groups", source: { name: "groups" } }], auth_type: "saml" }, app_launcher_visible: true, auto_redirect_to_identity: true }
     })

2. **Note the returned SSO metadata** from the response:
   - `idp_entity_id` — Cloudflare's Entity ID (configure in SaaS app)
   - `sso_endpoint` — Cloudflare's SSO URL (configure in SaaS app)
   - `public_key` — Certificate for SAML signature verification

#### B. Create an OIDC SaaS Application

1. **Create the SaaS app**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps`,
       body: { name: "Custom SaaS App", type: "saas", saas_app: { redirect_uris: ["https://app.example.com/callback"], grant_types: ["authorization_code"], scopes: ["openid", "email", "profile", "groups"], auth_type: "oidc" }, app_launcher_visible: true }
     })

2. **Note the returned OIDC config** from the response:
   - `client_id` — Configure in the SaaS app
   - `client_secret` — Configure in the SaaS app (returned once)

#### C. Add Policies to the SaaS App

3. **Create an Allow policy**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps/${appId}/policies`,
       body: { name: "Allow All Employees", decision: "allow", include: [{ group: { id: "<all-employees-group-id>" } }] }
     })

4. **Add a deny policy** for contractors (if needed):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps/${appId}/policies`,
       body: { name: "Deny Contractors", decision: "deny", include: [{ group: { id: "<contractors-group-id>" } }], precedence: 1 }
     })

#### D. Configure the SaaS App Side

5. In the SaaS app's admin console:
   - **SAML**: Enter Cloudflare's Entity ID, SSO URL, and certificate from step A.2.
   - **OIDC**: Enter the client_id and client_secret from step B.2. Set the issuer to Cloudflare's OIDC endpoint.

### Without MCP Tools (Dashboard/API)

1. Navigate to **Zero Trust → Access → Applications → Add an application → SaaS**.
2. Select the SaaS app from the catalog (or enter custom SAML/OIDC details).
3. Copy the provided SSO metadata (Entity ID, SSO URL, certificate) or OIDC credentials.
4. Configure the SaaS app's SSO settings with the copied values.
5. Add policies on the Access app to control who can access.
6. Test SSO flow by navigating to the SaaS app.

## Validate

1. **Verify Access app configuration**:
   → execute: GET `/accounts/${accountId}/access/apps/${appId}` — confirm SaaS app type, SAML/OIDC settings, allowed IdPs
   → execute: GET `/accounts/${accountId}/access/apps/${appId}/policies` — confirm policies exist with correct selectors

2. **Test SSO flow** (SP-initiated):
   - Navigate to the SaaS app's login page
   - Should redirect to Cloudflare Access → IdP login → back to SaaS app
   - Verify user attributes (groups, email) are correctly passed in the SAML assertion or OIDC token

3. **Test SSO flow** (IdP-initiated via App Launcher):
   - Navigate to `<team-name>.cloudflareaccess.com`
   - Click the SaaS app in the launcher
   - Should authenticate and land in the SaaS app

4. **Test deny scenario**:
   - User not matching any Allow policy → should see Access denied page
   - User matching a Deny policy → should see Access denied page

5. **Check access logs**:
   → execute: GET `/accounts/${accountId}/access/logs/access_requests` — look for authentication events for the SaaS app domain

## Operate

1. **SSO login fails**:
   - **Check IdP**: GET `/accounts/${accountId}/access/identity_providers` — is the IdP still configured and healthy?
   - **Check app config**: GET `/accounts/${accountId}/access/apps/${appId}` — verify SAML/OIDC settings haven't been changed
   - **Check logs**: GET `/accounts/${accountId}/access/logs/access_requests` — look for error details in denied events
   - **Common cause**: SaaS app's SSO config is stale (certificate rotated, URL changed)

2. **User can authenticate but lands on wrong SaaS tenant**:
   - SaaS federation doesn't control tenant selection — that's Gateway tenant control.
   - Use `gateway-dlp` or `gateway-policy-design` skills to configure HTTP tenant headers.

3. **Add new SaaS app**:
   - Follow Implement steps A or B above.
   - Remember to configure the SaaS app side with Cloudflare's SSO metadata.

4. **Rotate SAML certificate**:
   - Cloudflare manages the certificate. If it needs to be updated in the SaaS app, get the current cert from the Access app config.
   → GET `/accounts/${accountId}/access/apps/${appId}` — find `public_key` in `saas_app` config

5. **Audit SaaS access**:
   → GET `/accounts/${accountId}/access/logs/access_requests` — filter for SaaS app domains
   → GET `/accounts/${accountId}/access/users` — review which users have authenticated
   - Cross-reference with `casb-integration` for API-level posture scanning of the same SaaS apps.

## Gotchas

1. **SaaS apps federate authentication, not authorization** — Access controls WHO can log in. What the user can do inside the SaaS app is controlled by the SaaS app's own role/permission model.
2. **SAML attribute mapping is case-sensitive** — `Groups` ≠ `groups` in SAML assertions. Match the exact attribute name the SaaS app expects.
3. **OIDC client_secret is returned once** — When creating an OIDC SaaS app, the `client_secret` is returned only in the creation response. Store it immediately. If lost, you must recreate the app.
4. **App Launcher requires app_launcher_visible: true** — SaaS apps won't appear in the App Launcher unless this flag is set.
5. **IdP must be configured first** — SaaS federation requires an identity provider. Use `access-identity` skill to set up IdP and SCIM before creating SaaS apps.
6. **Tenant control is NOT part of SaaS federation** — To restrict which SaaS tenants users can access (e.g., only corporate Slack), use Gateway HTTP policies with tenant headers. SaaS federation only handles authentication.
7. **Multiple IdPs on SaaS apps** — If multiple IdPs are allowed, users see a selection screen. For a smoother experience with a single IdP, enable `auto_redirect_to_identity`.

## Related Skills

- **access-identity**: IdP must be configured before SaaS federation
- **access-app-setup**: Detailed app type guidance — SaaS is one of four Access app types
- **casb-integration**: API CASB complements SaaS federation with posture scanning
- **gateway-dlp**: Inline DLP scanning for SaaS uploads/downloads
- **gateway-policy-design**: Tenant control via Gateway HTTP policies
