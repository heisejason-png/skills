---
name: "access-private-network"
description: "Design private network access patterns using CF1 Client (WARP) and Cloudflare Tunnels. Covers split tunnel configuration, virtual networks for overlapping IP spaces, resolver policies, and Local Domain Fallback."
domain: "access"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/"
---

# Access Private Network Patterns

## Persona

You are a senior Cloudflare One architect specializing in private network access design. You understand the interplay between CF1 Client, tunnels, routes, split tunnels, and DNS resolution that makes private access work end-to-end.

## When to Use

- Customer is designing private network access for remote workers
- Customer has overlapping IP spaces across environments (needs virtual networks)
- Customer needs split tunnel configuration guidance
- Customer has DNS resolution issues for private hostnames
- Customer asks about the relationship between WARP, tunnels, and Access apps

## When NOT to Use

- Customer needs to create individual Access applications → use `access-app-setup`
- Customer needs to deploy the tunnel itself → use `tunnel-deployment`
- Customer needs site-to-site connectivity (not user-to-site) → use `wan-site-connectivity`

## Assess

1. **Map the network topology**:
   - What private networks/subnets need to be accessible? List CIDRs.
   - What private hostnames need to be resolvable? (e.g., `jira.corp.internal`, `gitlab.internal`)
   - Are there overlapping IP spaces across environments (dev/staging/prod)?

2. **Check existing tunnel connectivity**:
   - Is a Cloudflare Tunnel deployed to each target network? (Use `tunnel-deployment` skill if not)
   - What CIDR and hostname routes are configured on each tunnel?

3. **Check CF1 Client (WARP) deployment**:
   → execute: GET `/accounts/${accountId}/devices/policies` — check device settings profiles
   → execute: GET `/accounts/${accountId}/devices/policies/${profileId}/excludes` or `/includes` — check current split tunnel config
   - Is the CF1 Client deployed to user devices? What split tunnel mode is in use?

4. **Check existing Access apps for private resources**:
   → execute: GET `/accounts/${accountId}/access/apps` — look for apps with `type: "self_hosted"` and private destinations
   → execute: GET `/accounts/${accountId}/access/apps/${appId}/policies` — review policies on existing private apps

5. **Check identity and group configuration**:
   → execute: GET `/accounts/${accountId}/access/identity_providers` — confirm IdP is configured
   → execute: GET `/accounts/${accountId}/access/identity_providers/${idpId}/scim/groups` — confirm groups are available for policies
   → execute: GET `/accounts/${accountId}/access/groups` — check reusable groups

6. **Key questions**:
   - Which user groups need access to which networks?
   - Are there compliance requirements (device posture, geo restrictions)?
   - Is there a DNS architecture in place (internal DNS servers, split-horizon DNS)?

## Design

### Private Access Architecture

The private network access stack has four layers that must all be correctly configured:

| Layer | Component | Purpose |
|---|---|---|
| **On-ramp** | CF1 Client (WARP) | Routes user traffic to Cloudflare |
| **Identity** | Access app + policy | Controls who can reach the private resource |
| **Off-ramp** | Cloudflare Tunnel | Connects Cloudflare to the private network |
| **Routing** | CIDR/hostname routes | Tells Cloudflare which tunnel reaches which network |

### Split Tunnel Modes

- **Exclude mode** (default): All traffic routes to Cloudflare EXCEPT listed CIDRs/domains. Good for "protect everything" posture.
- **Include mode**: ONLY listed CIDRs/domains route to Cloudflare. Good when you want minimal traffic through Cloudflare.
- The split tunnel config on the device profile controls what the WARP client sends to Cloudflare. If a destination CIDR is excluded (in Exclude mode) or not included (in Include mode), WARP won't route it — and the Access app won't see the traffic.

### DNS Resolution for Private Hostnames

Private hostnames require **explicit DNS configuration** — they don't "just work." Two options:

1. **Resolver policy** (recommended): Configure in Gateway DNS settings. Points a domain (e.g., `*.corp.internal`) to a tunnel that has access to the internal DNS server. Requires Gateway DNS filtering to be active.
2. **Local Domain Fallback (LDF)**: Configured on the device profile. Sends DNS queries for specific domains to designated DNS servers. Works without Gateway DNS filtering but bypasses Gateway DNS logging.

### Virtual Networks for Overlapping IPs

When multiple environments share the same IP space (e.g., dev and prod both use `10.0.0.0/8`):
- Create a **virtual network** per environment
- Assign each tunnel to its virtual network
- Configure the CF1 Client device profile to select the active virtual network
- Access apps with private destinations are scoped to specific virtual networks

### WARP Auth Identity

- **Enable WARP auth identity** on private destination Access apps to bind device identity to the Access session.
- This means the WARP client proves the user's identity at the network level, not just the application level.
- Without WARP auth identity, a user could theoretically route traffic through WARP to a private IP without Access evaluating their identity for that specific resource.

## Implement

### With MCP Tools (Code Mode)

#### A. Create Private Destination Access Apps

1. **Create a private destination app** (IP/CIDR):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps`,
       body: { name: "Internal Jira", type: "self_hosted", session_duration: "24h", destinations: [{ type: "private", uri: "10.0.1.50:443" }], auto_redirect_to_identity: true, app_launcher_visible: true }
     })

2. **Create a private destination app** (hostname):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps`,
       body: { name: "Internal GitLab", type: "self_hosted", session_duration: "24h", destinations: [{ type: "private", uri: "gitlab.corp.internal:443" }], auto_redirect_to_identity: true }
     })

3. **Create a private destination app** (CIDR range — covers all services in a subnet):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps`,
       body: { name: "Dev Network", type: "self_hosted", session_duration: "12h", destinations: [{ type: "private", uri: "10.1.0.0/16" }] }
     })

#### B. Add Access Policies

4. **Create an Allow policy** on the app:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/access/apps/${appId}/policies`,
       body: { name: "Allow Engineering", decision: "allow", include: [{ group: { id: "<access-group-id>" } }], require: [{ device_posture: { integration_uid: "<disk-encryption-rule-id>" } }] }
     })

5. **Or attach a reusable policy**:
   → execute: GET `/accounts/${accountId}/access/policies` to find existing reusable policies
   - Attach to app via Dashboard or API.

#### C. Verify Split Tunnel Configuration

6. **Check current split tunnel config**:
   → execute: GET `/accounts/${accountId}/devices/policies` — identify the active profile
   → execute: GET `/accounts/${accountId}/devices/policies/${profileId}/excludes` (for Exclude mode) or `/includes` (for Include mode)
   - Verify that target private CIDRs are NOT in the exclude list (Exclude mode) or ARE in the include list (Include mode).

#### D. Verify Tunnel Routes

7. Ensure the off-ramp tunnel has CIDR or hostname routes for the target networks.
   → Use `tunnel-deployment` / `tunnel-routing` skills for tunnel route management.

### Without MCP Tools (Dashboard/API)

1. **Create private Access app**: Zero Trust → Access → Applications → Add application → Self-hosted → Add private destinations (IPs, CIDRs, or hostnames).
2. **Add policies**: On the app, add Allow policies with appropriate group/email/posture selectors.
3. **Check split tunnel**: Zero Trust → Settings → WARP Client → Device settings → select profile → Split Tunnels. Ensure target CIDRs are routed correctly.
4. **Configure DNS resolution**: Zero Trust → Gateway → DNS → Resolver policies (for resolver approach), or WARP Client → Device settings → Local Domain Fallback (for LDF approach).
5. **Virtual networks**: Zero Trust → Network → Virtual Networks → Create. Then assign tunnels and configure WARP client profile.

## Validate

1. **Verify Access app configuration**:
   → execute: GET `/accounts/${accountId}/access/apps/${appId}` — confirm destinations, session duration, WARP auth identity settings
   → execute: GET `/accounts/${accountId}/access/apps/${appId}/policies` — confirm policies exist with correct selectors and precedence

2. **Verify split tunnel**:
   → execute: GET `/accounts/${accountId}/devices/policies/${profileId}/excludes` or `/includes` — confirm target CIDRs are routed to Cloudflare

3. **Test connectivity** (end-to-end):
   - Connect CF1 Client on a test device
   - Attempt to reach private IP/hostname
   - For IP: `curl -v https://10.0.1.50:443` (or equivalent)
   - For hostname: `nslookup gitlab.corp.internal` (verify DNS resolves), then `curl -v https://gitlab.corp.internal`

4. **Test access control**:
   - Authorized user → should reach the resource
   - Unauthorized user → should see Access denied page
   - User without required device posture → should see posture failure

5. **Check access logs**:
   → execute: GET `/accounts/${accountId}/access/logs/access_requests` — look for events matching the private app domain/destination
   → execute: GET `/accounts/${accountId}/access/users/${userId}/active_sessions` — verify active session for the private app

6. **DNS resolution validation** (for hostname destinations):
   - On the test device with WARP active: `nslookup <private-hostname>`
   - Should resolve to the private IP via the resolver policy or LDF
   - If it resolves to a public IP or fails, DNS config is wrong

## Operate

1. **User can't reach private app**:
   - **Check CF1 Client**: Is WARP connected? Is the correct device profile active?
   - **Check split tunnel**: GET `/accounts/${accountId}/devices/policies/${profileId}/excludes` or `/includes` — is the destination CIDR excluded or not included?
   - **Check tunnel health**: Is the off-ramp tunnel healthy? (Use `tunnel-deployment` skill)
   - **Check routes**: Does the tunnel have a CIDR/hostname route for the destination?
   - **Check Access policy**: GET `/accounts/${accountId}/access/apps/${appId}/policies` — does the user match an Allow policy?
   - **Check logs**: GET `/accounts/${accountId}/access/logs/access_requests` — look for denied events
   → GET `/accounts/${accountId}/access/users/${userId}/failed_logins` — check for authentication failures

2. **DNS not resolving for private hostnames**:
   - Verify resolver policy exists in Gateway DNS settings for the domain
   - OR verify Local Domain Fallback is configured for the domain in the device profile
   - Check that the DNS server specified in the policy/LDF is reachable via the tunnel

3. **Overlapping IP conflicts**:
   - Verify virtual networks are configured
   - Verify each tunnel is assigned to the correct virtual network
   - Verify the device profile has the correct default virtual network selected

4. **Session management**:
   → GET `/accounts/${accountId}/access/users/${userId}/active_sessions` — check current sessions
   - To force re-authentication: POST `/accounts/${accountId}/access/apps/${appId}/revoke_tokens`

## Gotchas

1. **Private hostname resolution requires EXPLICIT DNS config** — resolver policy or Local Domain Fallback. This is the #1 support issue for private access. Hostnames don't resolve "magically" just because a tunnel exists.
2. **Split tunnel mode changes behavior fundamentally** — In Exclude mode, removing a CIDR from the exclude list sends it to Cloudflare. In Include mode, adding a CIDR to the include list sends it to Cloudflare. Mixing these up is the #2 support issue.
3. **Virtual networks must be assigned to BOTH the tunnel AND the device profile** — Assigning a virtual network to a tunnel but not the WARP client profile (or vice versa) results in unreachable destinations.
4. **WARP auth identity ≠ Access session** — WARP auth identity confirms "this device belongs to user X." Access session confirms "user X is authorized for app Y." Enable WARP auth identity on private destination apps to bind both.
5. **CIDR routes vs hostname routes** — IP/CIDR destinations use CIDR routes on the tunnel. Hostname destinations use hostname routes. Using the wrong route type means the tunnel won't serve the traffic.
6. **Default deny** — Access apps are default deny. A private destination Access app with no policies will block ALL access, even from devices with WARP connected. You must add at least one Allow policy.
7. **Gateway Network policies complement, don't replace** — Access apps provide per-app identity-aware control. Gateway Network policies provide broader network-level rules. Use both for defense-in-depth.

## Related Skills

- **tunnel-deployment**: Deploy the tunnel that provides connectivity to private networks
- **tunnel-routing**: Configure CIDR/hostname routes on the tunnel
- **access-app-setup**: Create Access applications — detailed app type guidance and policy design
- **access-identity**: SCIM provisioning for group-based Access policies
- **device-posture-setup**: Device posture rules that can be required in Access policies
- **gateway-network-policies**: Complementary network-level security for private traffic
