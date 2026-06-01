---
name: "access-app-setup"
description: "Create and configure Cloudflare Access applications for zero trust access to private or public resources. Covers all four application types (self-hosted public, self-hosted private, SaaS, infrastructure), policy design, and integration with tunnels, identity, and device posture."
domain: "access"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/applications/"
---

# Access Application Setup

## Persona

You are a senior Cloudflare One architect specializing in zero trust access control. You design Access applications that balance security posture with user experience, and you know the tradeoffs between the four application types cold.

## When to Use

- Customer wants to protect a web application, API, or infrastructure with identity-aware access
- Customer is setting up private application access for remote workers
- Customer is migrating from VPN or Zscaler ZPA to CF1 Access
- Customer needs SSO federation for a SaaS application
- Customer asks "how do I make [app] accessible only to authorized users"

## When NOT to Use

- Customer only needs DNS/HTTP content filtering → use `gateway-policy-design`
- Customer needs site-to-site network connectivity without per-app access control → use `wan-site-connectivity`
- Customer wants to set up the tunnel itself (not the Access app on top) → use `tunnel-deployment` first, then come back here

## Assess

Before designing an Access application, gather:

1. **What is the application?**
   - Web app, API, SSH/RDP target, database, or SaaS?
   - What protocol(s) and port(s)?
   - Is it on a public hostname or private IP/hostname?

2. **Where does it live?**
   - Public internet (already routable)?
   - Private network behind a tunnel?
   - Third-party SaaS (need SSO federation)?

3. **Who needs access?**
   - Which user groups/roles?
   - Internal employees only, or contractors/partners too?
   - Is an IdP already configured? Is SCIM provisioning active?

4. **What security controls are required?**
   - Device posture checks (OS, disk encryption, etc.)?
   - Geographic restrictions?
   - Session duration requirements?
   - mTLS client certificates?

5. **What connectivity exists?**
   - Is a Cloudflare Tunnel already deployed to the target network?
   - Is the CF1 Client (WARP) deployed to user devices?
   - Are there DNS resolver policies or Local Domain Fallback rules?

## Design

### Application Type Decision Matrix

This is the most important design decision. Get this wrong and you'll rebuild it.

| Scenario | App Type | Key Requirements |
|----------|----------|-----------------|
| Web app on a public hostname (e.g., `app.example.com`) | **Self-hosted (public hostname)** | DNS CNAME to tunnel if behind tunnel. Access sits on the hostname directly. Clientless — works in any browser. |
| Private app accessed by IP or private hostname (e.g., `10.0.1.50:8443` or `jira.corp.internal`) | **Self-hosted (private destination)** | Requires CF1 Client (WARP) on user devices OR network on-ramp. Needs complementary tunnel route (CIDR for IPs, hostname route for DNS names). WARP auth identity should be enabled. **This is the recommended model for identity-based access to private resources.** |
| Third-party SaaS (e.g., Salesforce, Slack, GitHub) | **SaaS** | Federated SSO. Access acts as an identity proxy. Requires IdP configuration. |
| SSH target with role-based auth | **Infrastructure** | SSH-specific with role-based auth policies. Less common — for most SSH access, prefer self-hosted (private destination) with Browser Rendering enabled. Infrastructure policies are NOT reusable across apps. |

### Critical Design Decisions

**Public hostname vs. private destination:**
- If users access via a friendly URL in a browser → public hostname (clientless)
- If users access via IP, non-HTTP protocol, or private DNS name → private destination (requires WARP)
- If the app is behind a tunnel but you want clientless browser access → public hostname with CNAME to tunnel ID (`<tunnel-id>.cfargotunnel.com`)

**Policy architecture:**
- Access apps are **default deny** — you must create at least one Allow policy
- Policies evaluate in **precedence order** (lower number = higher priority, first match wins)
- Use **reusable policies** when the same user groups access multiple apps
- Use **app-specific policies** for unique access requirements
- Only **IP lists** (not domain/URL lists) can be used as Access policy selectors

**Identity requirements:**
- SCIM provisioning is required for group-based policy selectors (`identity.groups.name`)
- Without SCIM, you can still use email, email domain, and service token selectors
- Multiple IdPs are supported — configure each in the Zero Trust dashboard
- For private destination apps, WARP auth identity ties the device identity to the Access session

**Private destination networking:**
- Each private destination needs a complementary route on the off-ramp:
  - IP/CIDR destinations → CIDR route on the tunnel
  - Private hostname destinations → Hostname route on the tunnel + DNS resolution (resolver policy or Local Domain Fallback)
- Virtual networks isolate overlapping IP spaces between environments
- Split tunnel configuration on the CF1 Client controls what traffic routes to Cloudflare

### When to Use Gateway Network Policies as Complement

Access apps provide identity-aware, per-application access control. Gateway Network policies provide broader network-level security. Use both when:
- You want a fallback "deny all private traffic" network policy with Access apps as explicit exceptions
- You need port/protocol filtering beyond what Access app destinations specify
- You want to log all private network traffic, not just Access-protected app traffic

Common pattern: Set an Access-complementary Gateway network rule at precedence 8000 ("Access Private Apps → Allow") and a default deny at a higher precedence number.

## Implement

### With MCP Tools (Code Mode)

**Step 1: Check existing applications**
```
→ search: "access applications" — find the endpoint for listing Access apps
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/access/apps` })
Review existing Access apps to avoid duplicates and understand current patterns.
```

**Step 2: Create the Access application**

For a self-hosted private destination app:
```
→ search: "create access application" — find the POST endpoint and request body schema
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/apps`,
    body: {
      name: "[app-name]",
      type: "self_hosted",
      session_duration: "24h",
      destinations: [{ type: "private", uri: "[10.0.1.0/24:8443 | jira.corp.internal]" }],
      policies: [],
      auto_redirect_to_identity: true,
      app_launcher_visible: true
    }
  })
```

For a self-hosted public hostname app:
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/apps`,
    body: {
      name: "[app-name]",
      type: "self_hosted",
      domain: "app.example.com",
      session_duration: "24h",
      policies: [],
      auto_redirect_to_identity: true,
      app_launcher_visible: true
    }
  })
```

**Step 3: Create Access policies**
```
→ search: "create access application policy" — find POST endpoint for app-specific policies
→ execute (app-specific policy):
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/apps/${appId}/policies`,
    body: {
      name: "Allow [group-name]",
      decision: "allow",
      include: [{ group: { id: "[access-group-uuid]" } }],
      require: [{ device_posture: { integration_uid: "[posture-rule-uuid]" } }]
    }
  })

→ execute (reusable policy):
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/policies`,
    body: { /* same structure */ }
  })
```

**Step 4: Verify tunnel routes exist (for private destinations)**
Ensure the target network has CIDR or hostname routes pointing to the appropriate tunnel.
→ Related skill: `tunnel-deployment`

### Without MCP Tools (Dashboard)

1. **Dashboard**: Zero Trust → Access → Applications → Add an application
2. Select application type based on the decision matrix above
3. Configure:
   - Name and logo
   - Domain (public hostname) or Destinations (private IPs/hostnames)
   - Session duration (recommended: 24h for standard, 8h for sensitive apps)
   - Identity providers (select which IdPs can authenticate)
4. Add policies:
   - Click "Add a policy"
   - Name it descriptively (e.g., "Allow Engineering Team")
   - Action: Allow
   - Include rules: Select group, email domain, or other selectors
   - Require rules (optional): Device posture, country, etc.
5. Configure advanced settings:
   - CORS (if API access needed from browsers)
   - Cookie settings (SameSite, HttpOnly)
   - Browser rendering (enable for SSH/VNC/RDP applications)

### Without MCP Tools (API)

```bash
# Create a self-hosted private destination app
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/access/apps" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Internal Jira",
    "type": "self_hosted",
    "session_duration": "24h",
    "destinations": [{"type": "private", "uri": "jira.corp.internal:443"}],
    "auto_redirect_to_identity": true
  }'

# Create an Allow policy on the app
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/access/apps/{app_id}/policies" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Allow Engineering",
    "decision": "allow",
    "include": [{"group": {"id": "{group_uuid}"}}],
    "precedence": 1
  }'
```

## Validate

1. **Test access with an authorized user:**
   - For public hostname: Navigate to the app URL → should see IdP login → access granted
   - For private destination: Connect CF1 Client → navigate to private IP/hostname → access granted

2. **Test access with an unauthorized user:**
   - User outside the policy group → should see Access blocked page
   - User without required device posture → should see posture failure message

3. **Verify session behavior:**
   - After session duration expires, user should be re-prompted for authentication
   - Check Access audit logs for successful and denied sessions

4. **Check edge cases:**
   - Multiple IdP scenario: Can users from each configured IdP authenticate?
   - Service token access (if configured): Can automated systems authenticate?
   - Private destination DNS: Does the private hostname resolve correctly through the tunnel?

5. **With MCP Tools (Code Mode):**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/access/apps/${appId}` })
  — verify app configuration matches design
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/access/apps/${appId}/policies` })
  — verify policies are in correct precedence order
```

## Operate

### Monitoring
- **Access audit logs**: Review authentication successes/failures regularly
- **Session counts**: Monitor concurrent sessions for capacity planning
- **Policy hit rates**: Identify policies that never match (possible misconfiguration)

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| "Access denied" for authorized users | Policy precedence wrong, or group name mismatch (case-sensitive with SCIM) | Check policy order; verify IdP group names match exactly |
| Private app unreachable | Tunnel down, missing route, CF1 Client disconnected, or split tunnel excluding destination | Verify tunnel health, check CIDR/hostname routes, confirm WARP connection |
| Session expiry too aggressive | Session duration too short | Adjust duration; consider Instant Auth for low-risk apps |
| CORS errors on API calls | CORS not enabled in Access app settings | Enable CORS; set allowed origins, methods, headers |
| Browser Rendering not working | Feature not enabled on the app, or wrong app type | Enable Browser Rendering in app settings; must be self-hosted type |

### When to Revisit
- New user groups need access → add or update policies
- App moves to a different network → update tunnel routes and Access destinations
- Compliance audit requires MFA enforcement → add device posture requirement to policies
- Scaling past 1000+ Access apps → consider reusable policies and Access Groups for manageability

## Gotchas

1. **IP lists only in Access policies** — Domain lists and URL lists are Gateway-only constructs. Access policies can only reference IP lists. This trips up migration engineers constantly.

2. **Infrastructure app policies are NOT reusable** — Unlike self-hosted and SaaS apps, infrastructure application policies are linked directly to the app. If you need the same policy on multiple SSH targets, use self-hosted private destination apps instead.

3. **Private hostname resolution requires explicit DNS config** — Private hostnames don't "just work." You need either a resolver policy pointing the domain to a tunnel, or Local Domain Fallback configured on the CF1 Client. This is the #1 support issue for private app deployments.

4. **WARP auth identity is not the same as Access session** — These are related but distinct. WARP auth identity confirms the device has a registered user. Access session confirms the user is authorized for a specific app. For private destination apps, enable WARP auth identity to bind both together.

5. **Reusable policies precedence** — When a reusable policy is attached to an app, it evaluates at the precedence set on the attachment, not the policy's own precedence. This can cause unexpected ordering if not managed carefully.

6. **Browser Rendering for SSH/VNC** — Self-hosted apps with Browser Rendering enabled provide a web-based SSH/VNC client without requiring the CF1 Client. Prefer this for contractor/partner access where you can't mandate the client.

7. **Legacy cloudflared access proxy** — Still works for SSH/SMB but is the old pattern. For new deployments, prefer CF1 Client + private destination routing. The legacy proxy doesn't support all Access features.

8. **`auto_redirect_to_identity`** — When enabled, users are immediately redirected to the IdP instead of seeing the Access login page. Good UX for single-IdP setups, confusing for multi-IdP (user doesn't get to choose).

## Related Skills

- **tunnel-deployment**: Set up the Cloudflare Tunnel that provides connectivity to private applications. Must be done before creating private destination Access apps.
- **access-identity**: Configure IdP integration and SCIM provisioning. Must be done before creating group-based Access policies.
- **gateway-policy-design**: Set up complementary Gateway Network policies for defense-in-depth on private traffic.
- **device-posture-setup**: Configure device posture rules that can be required in Access policies.
- **migrate-zscaler-zpa**: If migrating from ZPA, this skill handles the mapping of app segments → Access apps and connector groups → tunnels.
