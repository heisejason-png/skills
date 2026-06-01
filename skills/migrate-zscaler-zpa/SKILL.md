---
name: "migrate-zscaler-zpa"
description: "Migrate from Zscaler Private Access (ZPA) to Cloudflare One Access + Tunnels. Covers app segment → Access app mapping (standard/agentless/IP-anchored), connector group → tunnel mapping, server group → CIDR/hostname routes, access policy migration with SCIM/posture handling, bypass → split tunnel, and resolver policy generation."
domain: "migration"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/"
---

# Migrate Zscaler ZPA to Cloudflare One

## Persona

You are a senior Cloudflare One migration architect with deep knowledge of both Zscaler Private Access (ZPA) and CF1 Access + Tunnels. You understand that ZPA's 3-way binding model (app segment ↔ server group ↔ connector group) does **not** map 1:1 to CF1 — it requires architectural restructuring where tunnels provide routes and Access apps define destinations independently.

## When to Use

- Customer is migrating from Zscaler ZPA to CF1 Access + Tunnels
- Customer has a ZPA JSON config export (from ZPA Admin Portal → API or Expedition)
- Customer asks about ZPA-to-CF1 feature mapping or competitive assessment
- Pre-sales needs a migration scope estimate (app segment count, connector groups, identity complexity)

## When NOT to Use

- Customer is migrating from ZIA (internet security) → use `migrate-zscaler-zia`
- Customer only needs Access app setup without migration context → use `access-app-setup`
- Customer is migrating from Palo Alto → use `migrate-palo-alto`
- Greenfield CF1 deployment with no existing ZPA config → use product-specific skills directly

## Assess

### 1. Obtain the ZPA Configuration Export

The migration requires a ZPA JSON export containing structured data. Export from the ZPA Admin Portal API or use Zscaler's Expedition tool.

**Required keys in the export:**
- `application` — app segments (domains, ports, segment groups, server group links)
- `app_connector_group` — connector groups (location, connectors)
- `server_group` — server groups (linked connector groups, servers, applications)
- `server` — individual servers (IP addresses)
- `segment_group` — segment groups (application grouping)
- `policyset_rules_access_policy` — access policy rules (conditions, actions)
- `posture` — posture profiles (type, domain)

**Optional keys:**
- `policyset_client_forwarding_policy` — client forwarding rules (agentless app detection)
- `policyset_bypass_policy` — bypass rules (split tunnel candidates)
- `policyset_isolation_policy` — isolation rules (RBI tagging)
- `policyset_timeout_policy` / `policyset_reauth_policy` — session policies
- `connector` — individual connector instances
- `auth_domains`, `machine_group`, `platform`, `client_types` — supplementary config

### 2. Gather Migration Options

| Option | Default | Purpose |
|--------|---------|---------|
| `prefix` | `descaler_` | Prefix for all generated CF1 object names. Enables identification and cleanup. |
| `disabled` | `true` | Generated rules/apps start disabled. **Always start disabled** — review before enabling. |
| `scim_configured` | `false` | Is SCIM provisioning active? If `false`, SAML group/SCIM group references become description annotations only. |
| `posture_configured` | `false` | Are device posture checks configured in CF1? If `false`, posture references tagged `[needs_posture]`. |
| `create_gateway_ordering_rule` | `true` | Create a Gateway L4 rule (`Access Private Apps → Allow`) to prevent Gateway from blocking Access-protected traffic. |
| `ip_anchoring_mode` | `tunnel_egress` | How to handle IP-anchored apps: `tunnel_egress` (preserve corporate IP via tunnel) or `egress_policy` (use CF dedicated egress IPs). |
| `resolver_entries` | `[]` | Manual DNS resolver entries: `[{pattern: "*.corp.internal", dns_server: "10.1.1.53"}]`. |

### 3. Assess Scope Complexity

Before designing, quantify:

- **App segments** — total count, how many disabled, how many IP-anchored, how many agentless
- **Connector groups** — each becomes a Cloudflare Tunnel; count locations and connector instances
- **Server groups** — bridge between apps and connectors; count servers (IP-based → CIDR routes)
- **Access policy rules** — identity conditions (SAML groups, SCIM groups, email, country), posture requirements
- **Posture profiles** — types (domain join, disk encryption, firewall, OS version, AV, certificate)
- **Bypass rules** — apps that should be split-tunnel excluded
- **Isolation policies** — apps requiring Remote Browser Isolation (RBI)
- **Internal hostnames** — domains needing resolver policies for private DNS resolution

## Design

### Architectural Difference: ZPA vs CF1

This is the most important concept for the migration. ZPA and CF1 use fundamentally different models:

```
ZPA Model (3-way binding):
  App Segment ←→ Server Group ←→ Connector Group
  (what)          (where)          (how)
  
  App Segment defines: domains, ports, segment group
  Server Group defines: which servers, which connector groups
  Connector Group defines: physical connectors at a site

CF1 Model (independent layers):
  Access Application → defines destinations (IPs, hostnames, ports)
  Cloudflare Tunnel  → provides routes (CIDR routes, hostname routes)
  They meet at the IP/hostname level — no explicit binding
```

The migration **restructures** the 3-way binding into independent CF1 components that connect at the network layer.

### Transform Pipeline Overview

The ZPA → CF1 migration is a 7-stage pipeline:

```
Stage 1: Connectivity   — connector groups → Tunnels, servers → CIDR routes, app domains → hostname routes
Stage 2: Resolver        — user entries + auto-detected internal hostnames → resolver policies
Stage 3: Applications    — app segments → Access apps (standard, agentless, IP-anchored)
Stage 4: Posture         — ZPA posture profiles → CF posture check name mapping
Stage 5: Policies        — ZPA access rules → CF Access policies + reusable Access groups
Stage 6: Bypass          — bypass rules → split tunnel exclude recommendations
Stage 7: Gateway Order   — single Gateway L4 rule: "Access Private Apps → Allow"
```

### Stage 1: Connectivity (Connector Groups → Tunnels + Routes)

**One Cloudflare Tunnel per connector group.** Each ZPA connector group becomes a named tunnel.

| ZPA Construct | CF1 Construct | Notes |
|---|---|---|
| Connector group | Cloudflare Tunnel | Named `<prefix><sanitized_group_name>` |
| Connectors (N instances) | cloudflared instances | Customer must install cloudflared on each host |
| Server (IP address) in server group | CIDR route on tunnel | Single IPs become /32 routes |
| App domain in server group | Hostname route on tunnel | Domain-to-tunnel mapping |

**Deduplication:** CIDR and hostname routes are deduped by `(network/hostname, tunnel_name)` — the same route won't appear twice on the same tunnel.

**Warnings generated:**
- Disabled connector groups → tunnel created but noted in description
- Connector groups with no linked server groups → tunnel has no routes
- Orphaned servers not in any server group → no routes created

### Stage 2: Resolver Policies

Resolver policies handle private DNS resolution for internal hostnames that the tunnel's upstream DNS can't resolve.

**Two sources:**
1. **User-provided entries** — explicit `{pattern, dns_server}` pairs (e.g., `*.corp.internal → 10.1.1.53`)
2. **Auto-detected internal hostnames** — app segment domains matching internal patterns (`.internal.`, `.corp.`, `.local.`, `.lan.`, `.private.`, single-label names, non-standard TLDs)

Hostnames already covered by user-provided patterns (including wildcard matching) are excluded. Uncovered internal hostnames generate a `manual_review` warning.

> CF resolver policies are **account-wide**, not per-tunnel. A single resolver policy applies to all tunnels.

### Stage 3: Application Classification

Each ZPA app segment is classified into one of three types:

| Type | Detection | CF1 Output |
|---|---|---|
| **Standard (private)** | Default — not agentless, not IP-anchored | Access app with private destinations (hostnames + ports). WARP client required. |
| **Agentless (clientless)** | Client forwarding policy has `CLIENT_TYPE=zpn_client_type_browser_isolation` for this app | Access app with public hostname + tunnel ingress route. One app per domain. No WARP required. |
| **IP-anchored (egress)** | `ipAnchored=true` on the app segment | Access app with hostname destinations. Egress mode choice: `tunnel_egress` or `egress_policy`. |

**Destination splitting:** Access apps are limited to 5 destinations per app (`MAX_DESTINATIONS_PER_APP`). Apps with more destinations are automatically split into multiple Access apps with identical policies (e.g., `AppName (1)`, `AppName (2)`).

**Agentless apps:**
- One Access app created per domain (not per app segment)
- Each domain gets a tunnel ingress route: `hostname → service` (e.g., `app.corp.com → http://localhost:443`)
- RDP apps detected by port 3389 → ingress uses `rdp://localhost:3389`

**IP-anchored apps (two modes):**

| Mode | Behavior | Trade-off |
|---|---|---|
| `tunnel_egress` | Traffic egresses through tunnel at customer site, preserving corporate source IP | Requires tunnel connector with internet egress at the site |
| `egress_policy` | Traffic egresses through Cloudflare dedicated egress IPs | Simpler, but source IP changes — target services must be updated to accept new IPs |

**Isolation tagging:** Apps targeted by ZPA isolation policies are tagged `[isolation_required]`. CF1 implements this via Access app browser rendering (RBI) settings.

### Stage 4: Posture Mapping

ZPA uses a **check-then-defer** model — posture profile IDs are mapped to names, but actual CF posture rules are NOT auto-created. Instead:

| Posture Configured? | Behavior |
|---|---|
| **Yes** | ZPA posture profile names referenced directly in Access policy `require` conditions. User must ensure CF posture check names match exactly. |
| **No** | Access policies referencing posture are tagged `[needs_posture]`. No posture conditions added. |

**ZPA posture type → CF equivalent:**

| ZPA Type | CF Posture Type |
|---|---|
| `DOMAIN` | `domain_joined` |
| `FIREWALL` | `firewall` |
| `DISK_ENCRYPTION` | `disk_encryption` |
| `ANTIVIRUS` | `application` |
| `OS_VERSION` | `os_version` |
| `CERTIFICATE` | `client_certificate` |

### Stage 5: Access Policy Migration

Each ZPA access policy rule maps to a CF Access policy attached to specific Access apps.

**Target app resolution:** ZPA rules reference apps via `APP` (direct app ID) or `APP_GROUP` (segment group ID → all apps in that group). These are resolved to the CF Access app names generated in Stage 3 (including split app names).

**Condition mapping:**

| ZPA Condition Type | CF Access Condition | Notes |
|---|---|---|
| `SAML` (memberOf/groups) | `include: [{type: "group", value: "<name>"}]` | Only if SCIM configured; otherwise `needs_identity` tag |
| `SAML` (department) | `include: [{type: "group", value: "<name>"}]` | Same SCIM dependency |
| `SAML` (email) | `include: [{type: "email", value: "<addr>"}]` | Direct mapping |
| `SAML` (email_domain) | `include: [{type: "email_domain", value: "<domain>"}]` | Direct mapping |
| `SCIM_GROUP` | `include: [{type: "group", value: "<name>"}]` | Only if SCIM configured |
| `POSTURE` | `require: [{type: "device_posture", value: "<id>"}]` | Only if posture configured |
| `COUNTRY_CODE` | `include: [{type: "country", value: "<code>"}]` | Direct mapping |
| `PLATFORM` | Warning only | CF Access has no direct platform condition — use device posture instead |
| `RISK_FACTOR_TYPE` | Warning (unsupported) | Consider external evaluation rules |

**Action mapping:** ZPA `ALLOW` → CF `allow`, `DENY` → `deny`, `BYPASS` → `bypass`.

**Access group deduplication:** When 3+ policies share identical condition sets (same include/exclude/require), a reusable Access Group is created to reduce duplication.

### Stage 6: Bypass → Split Tunnel

ZPA bypass rules identify apps that should bypass the ZPA tunnel. In CF1, these map to **WARP split tunnel exclude** entries — domains/IPs excluded from the WARP tunnel.

The transform emits `manual_review` warnings listing bypassed apps. The customer must manually add their domains to the WARP split tunnel exclude list.

Client forwarding rules with `CLIENT_TYPE=zpn_client_type_browser_isolation` are **not** bypass — they identify agentless apps (consumed by Stage 3).

### Stage 7: Gateway Ordering Rule

A single Gateway Network L4 rule is created at precedence **8000**:

```
Name: <prefix>Access Private Apps — Allow
Traffic: any(access.private_app[*] in {"*"})
Action: allow
Filter: l4
Precedence: 8000
```

This ensures traffic destined for Access private apps is not blocked by Gateway network policies (especially important when ZIA-migrated block rules exist at precedence 6000+).

### Design Decision Matrix

| Scenario | Recommendation | Rationale |
|---|---|---|
| ZPA app with WARP-only access | Standard private Access app | Direct mapping — WARP client handles tunnel routing |
| ZPA app with browser-based access | Agentless Access app (public hostname) | One app per domain + tunnel ingress route |
| ZPA IP-anchored app, customer needs same source IP | `tunnel_egress` mode | Traffic exits at customer site, preserving corporate IP |
| ZPA IP-anchored app, customer OK with new IP | `egress_policy` mode | Simpler — no tunnel egress needed, but target services must accept new IPs |
| App with >5 destinations | Auto-split into N apps | Each chunk gets identical policies; warn about the split |
| SCIM not configured | Migrate anyway, tag `needs_identity` | Generate the structure; fill in identity after SCIM setup |
| Posture not configured | Migrate anyway, tag `needs_posture` | Generate the structure; fill in posture after integration setup |
| Internal hostnames without resolver entries | Add resolver policies | Auto-detect flags uncovered hostnames for manual DNS config |
| Dual ZIA + ZPA migration | Run both transforms, enable gateway ordering rule | Gateway ordering rule at 8000 prevents ZIA network blocks from hitting ZPA private apps |

## Implement

This section provides **three implementation paths** for applying the migration design. Choose based on tooling access:

| Path | When to Use | Prerequisites |
|------|------------|---------------|
| **Automated (Descaler)** | Full migration with many app segments. Handles the entire 7-stage pipeline in one pass. | Access to a Descaler deployment |
| **MCP Tools** | Incremental migration, or when Descaler is unavailable. You drive each stage using CF1 MCP tools. | MCP server connected with Access, Tunnels, Gateway, and Devices apps |
| **Dashboard / API** | No tooling available, or for manual review of individual components. | CF dashboard access or API token with `Zero Trust: Edit` scope |

All three paths use the **same migration options** from Assess § 2. When using MCP or Dashboard/API, apply options manually at each step — prepend `prefix` to every name, set apps/rules disabled when `disabled=true`, include identity conditions only when `scim_configured=true`.

### Automated Migration (Descaler)

> **Descaler** is a Cloudflare Workers app for automated migration transforms. Deploy your own instance with `npm run build && npm run deploy`.

1. Open the Descaler UI in your browser
2. Select **"Zscaler ZPA"** as the source vendor
3. Upload the ZPA JSON export
4. Configure migration options in the UI:
   - **Prefix** — object name prefix (default: `descaler_`)
   - **Start disabled** — toggle on (recommended)
   - **SCIM configured** — toggle on only if SCIM is live
   - **Posture configured** — toggle on only if device posture checks exist in CF1
   - **IP anchoring mode** — `tunnel_egress` or `egress_policy`
   - **Gateway ordering rule** — toggle on (recommended for dual ZIA+ZPA)
   - **Resolver entries** — add `{pattern, dns_server}` pairs for internal DNS
5. Click **Transform** → review output and all warnings
6. Choose export format:
   - **JSON** — raw `CloudflareConfig` for inspection or custom tooling
   - **Terraform** — HELIX-compatible `.tf` file using `var.account_id`
   - **Python script** — standalone `deploy.py` with dry-run support
   - **Direct API** — browser-based import via CF API proxy with progress tracking
7. **Review all warnings** before deploying — see Validate § 1

### With MCP Tools (Code Mode)

Execute in dependency order: Tunnels → Routes → Resolver → Access apps → Access policies → Posture → Gateway ordering.

**Step 1: Create Cloudflare Tunnels (one per connector group)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/cfd_tunnel`,
    body: {
      name: "<prefix><sanitized_connector_group_name>",
      config_src: "cloudflare"
    }
  })
```
> Tunnels are created via API but **tokens are NOT returned**. Customer must: (1) install cloudflared on each connector host, (2) run `cloudflared tunnel login`, (3) configure routes, (4) start tunnel with `cloudflared tunnel run`.

**Step 2: Create CIDR routes (from servers)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/teamnet/routes`,
    body: {
      tunnel_id: "<tunnel_id>",
      network: "10.1.1.10/32",
      comment: "Server \"<server_name>\" in group \"<server_group_name>\""
    }
  })
```

**Step 3: Create Access Applications (standard private)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/apps`,
    body: {
      name: "<prefix><app_segment_name>",
      type: "self_hosted",
      session_duration: "24h",
      destinations: [{ type: "private", uri: "<hostname>:<port>" }]
    }
  })
```

For agentless apps, use public hostname type:
```
body: {
  name: "<prefix><app_name> — <domain>",
  type: "self_hosted",
  domain: "<domain>",
  session_duration: "24h"
}
```
Then configure the tunnel ingress route to point `<domain>` → `http://localhost:<port>` (or `rdp://localhost:3389` for RDP apps).

For apps with >5 destinations, split into multiple apps:
```
"<prefix><app_name> (1)" — destinations 1-5
"<prefix><app_name> (2)" — destinations 6-10
```

**Step 4: Create Access Policies (from ZPA access rules)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/apps/${appId}/policies`,
    body: {
      name: "<prefix><rule_name>",
      decision: "allow",
      include: [{ group: { name: "<SAML_group_name>" } }],
      require: [{ device_posture: { integration_uid: "<posture_check_id>" } }]
    }
  })
```

If SCIM is not configured, use `everyone` include and document groups in description:
```
body: {
  name: "<prefix><rule_name>",
  decision: "allow",
  include: [{ everyone: {} }],
  description: "SAML group \"<group_name>\" referenced — configure SCIM and update. Tag: [needs_identity]"
}
```

**Step 5: Create Gateway Ordering Rule**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix>Access Private Apps — Allow",
      description: "Allow traffic to Access private applications. Prevents Gateway network policies from blocking private app traffic.",
      action: "allow",
      traffic: "any(access.private_app[*] in {\"*\"})",
      filters: ["l4"],
      precedence: 8000,
      enabled: false
    }
  })
```

**Step 6: Create Device Posture Rules (if posture_configured)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/devices/posture`,
    body: {
      name: "<zpa_posture_profile_name>",
      type: "disk_encryption",
      description: "Migrated from ZPA posture profile \"<name>\"",
      match: [{ platform: "windows" }, { platform: "mac" }]
    }
  })
```

### Without MCP Tools (Dashboard/API)

**Tunnels:**
1. Dashboard: Zero Trust → Networks → Tunnels → Create a tunnel
2. Name with prefix, note the connector group source
3. Install cloudflared on connector hosts, configure routes

**API equivalent:**
```bash
# Create a tunnel
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/cfd_tunnel" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{"name": "descaler_dc-east-connectors", "tunnel_secret": "<base64_secret>"}'

# Add a CIDR route
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/teamnet/routes" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{"network": "10.1.1.10/32", "tunnel_id": "<tunnel_id>", "comment": "Server in group DC-East"}'
```

**Access Applications:**
1. Dashboard: Zero Trust → Access → Applications → Add an application → Self-hosted
2. For standard apps: configure private destinations (IPs/hostnames + ports)
3. For agentless apps: configure public hostname + tunnel ingress
4. Add policies with identity conditions

**Device Posture:**
1. Dashboard: Zero Trust → Settings → WARP Client → Device posture
2. Create checks matching ZPA posture profile types
3. Name them to match exactly — Access policies reference by name

**Gateway Ordering Rule:**
1. Dashboard: Zero Trust → Gateway → Firewall Policies → Network
2. Add rule: `any(access.private_app[*] in {"*"})` → Allow, precedence 8000

**Resolver Policies:**
1. Dashboard: Zero Trust → Gateway → Resolver policies
2. Add entries for internal hostnames: pattern → upstream DNS server

**Split Tunnel (Bypass):**
1. Dashboard: Zero Trust → Settings → WARP Client → Split Tunnels
2. Add domains from ZPA bypass rules to the exclude list

## Validate

### 1. Review Warnings

Every migration produces warnings. Review them **before enabling any apps or rules**:

- **`manual_review`** — Tunnels with no routes, apps with no destinations, policy rules referencing no apps, uncovered internal hostnames, IP-anchored egress mode guidance, bypass rules needing split tunnel config, tunnel install instructions
- **`partial`** — Disabled ZPA objects migrated with tags, apps split due to destination limits, SAML attributes with no direct CF mapping, client forwarding rules detected, posture profiles with suggested CF equivalents
- **`unsupported`** — Risk factor conditions, unknown condition types, platform restrictions (no direct CF Access equivalent)

### 2. Verify Object Counts

Check that the migration produced the expected number of objects:

```
→ execute: GET `/accounts/${accountId}/cfd_tunnel`             — verify one tunnel per connector group
→ execute: GET `/accounts/${accountId}/access/apps`            — verify Access applications (including split apps)
→ execute: GET `/accounts/${accountId}/access/policies`        — verify reusable policies
→ execute: GET `/accounts/${accountId}/access/groups`          — verify deduped Access groups
→ execute: GET `/accounts/${accountId}/devices/posture`        — verify posture rules (if configured)
→ execute: GET `/accounts/${accountId}/gateway/rules`          — verify gateway ordering rule (filter l4)
```

Cross-reference with pipeline output summary: tunnels created, CIDR routes, hostname routes, Access apps (standard/agentless/IP-anchored), Access policies, Access groups, warnings.

### 3. Validate Connectivity

For each tunnel:
1. Verify cloudflared is installed and running on connector hosts
2. Test CIDR route reachability from a WARP-enrolled device
3. For agentless apps: verify tunnel ingress routes resolve correctly (hostname → service)
4. For hostname routes: verify private DNS resolution via resolver policies

### 4. Test Access Policies

Enable apps one at a time:
1. **Standard private apps first** — test with WARP client, verify destination reachability
2. **Agentless apps** — test in browser without WARP, verify public hostname access
3. **IP-anchored apps** — verify egress IP (corporate or CF dedicated) matches expectations
4. **Identity policies** — test with users in referenced SAML/SCIM groups
5. **Posture policies** — verify device compliance before enforcing

### 5. Edge Case Checks

- Apps with >5 destinations were split — verify all parts have the same policies attached
- Disabled ZPA apps tagged `[disabled]` — verify they should remain disabled in CF1
- Apps tagged `[isolation_required]` — verify RBI browser rendering is configured on those Access apps
- Apps tagged `[needs_identity]` — placeholder until SCIM is configured
- Apps tagged `[needs_posture]` — placeholder until posture checks are created

## Operate

### Monitoring

- **Access audit logs**: Monitor authentication for migrated apps (filter by prefix)
- **Tunnel status**: Check connector health, route availability
- **Gateway activity logs**: Verify gateway ordering rule is matching private app traffic
- **Device posture status**: Check compliance rates for migrated posture checks
- **Resolver policy logs**: Verify internal hostname resolution is working

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Access app unreachable via WARP | Missing CIDR or hostname route on tunnel | Verify tunnel has routes for the app's destinations |
| Agentless app shows 502 error | Tunnel ingress route misconfigured | Check `service` field (protocol + port) matches the backend app |
| Identity policy not enforcing | SCIM group name mismatch | Verify group names match exactly (case-sensitive) between IdP and CF1 |
| Posture check failing all devices | CF posture rule name doesn't match ZPA profile name | Rename CF posture check to match exactly |
| Private hostname not resolving | Missing resolver policy | Add resolver entry: `hostname_pattern → internal_dns_server_ip` |
| IP-anchored app showing wrong source IP | Wrong egress mode selected | Switch between `tunnel_egress` and `egress_policy` as needed |
| Gateway blocking private app traffic | Gateway ordering rule missing or disabled | Create/enable the `Access Private Apps → Allow` rule at precedence 8000 |
| Split app missing policies | Policies not attached to all split parts | Verify `target_app_names` includes all `(1)`, `(2)`, etc. parts |

### When to Revisit

- **Post-SCIM setup**: Revisit all Access policies tagged `[needs_identity]` — replace `everyone` include with SCIM group conditions
- **Post-posture setup**: Revisit all Access policies tagged `[needs_posture]` — add device posture `require` conditions
- **Tunnel consolidation**: After validation, consider merging tunnels serving the same site/network
- **Destination limit changes**: If CF raises the per-app destination limit above 5, consider merging split apps
- **Bypass cleanup**: Periodically review split tunnel excludes — remove entries for decommissioned ZPA apps

## Gotchas

1. **ZPA's 3-way binding doesn't exist in CF1.** ZPA binds app segments ↔ server groups ↔ connector groups. CF1 has independent tunnels (with routes) and Access apps (with destinations). The migration restructures this — it is NOT a 1:1 mapping. Understand this before reviewing output.

2. **Tunnel tokens are NOT returned by the API.** Creating a tunnel via API or MCP gives you a tunnel ID but not the authentication token. The customer must install cloudflared on each connector host and authenticate manually (`cloudflared tunnel login`). Plan for connector deployment as a separate workstream.

3. **Apps are split at 5 destinations.** Any ZPA app segment with >5 domains/IPs produces multiple CF1 Access apps (e.g., `AppName (1)`, `AppName (2)`). All split parts get identical policies. Verify this after migration.

4. **Agentless apps produce one Access app PER DOMAIN**, not per app segment. A ZPA agentless app with 10 domains produces 10 CF1 Access apps. Each gets its own tunnel ingress route.

5. **IP-anchored apps require an egress mode choice.** `tunnel_egress` preserves the corporate source IP (traffic exits at the customer site via tunnel) but requires the connector host to have internet egress. `egress_policy` uses Cloudflare dedicated egress IPs — simpler but the target service must accept the new IPs. This is a **customer decision**, not an automated one.

6. **SAML group references require SCIM to be enforceable.** Without SCIM, ZPA SAML `memberOf`/`groups` conditions produce Access policies with `include: [everyone]` and groups noted in the description. Tagged `[needs_identity]`. These are placeholders, NOT enforcing identity.

7. **Posture is check-then-defer, NOT auto-created.** The migration does NOT create CF device posture rules. It maps ZPA posture profile IDs to names and expects the customer to have matching CF posture checks already created. If posture is not configured, policies are tagged `[needs_posture]`.

8. **ZPA `PLATFORM` conditions have no direct CF Access equivalent.** ZPA can restrict by platform (Windows, Mac, iOS, Android). CF Access does not have a platform selector — the migration emits a warning suggesting device posture checks (OS version) as an alternative.

9. **ZPA `RISK_FACTOR_TYPE` is unsupported.** Risk-based conditions have no CF Access mapping. Consider implementing via external evaluation rules.

10. **Bypass rules → split tunnel, NOT Access bypass.** ZPA bypass rules mean "don't route through ZPA." In CF1, this maps to WARP split tunnel excludes (domains/IPs excluded from the tunnel), NOT to Access `bypass` decisions. The migration emits warnings but does NOT auto-configure split tunnels.

11. **Client forwarding rules with `zpn_client_type_browser_isolation` are agentless detection, not RBI.** Despite the ZPA naming, `zpn_client_type_browser_isolation` identifies clientless/agentless access, NOT Remote Browser Isolation. Actual RBI is detected from `policyset_isolation_policy`.

12. **Resolver policies are account-wide.** A resolver policy for `*.corp.internal → 10.1.1.53` applies to ALL tunnels in the account, not just the one serving that network. Be careful with overlapping resolver entries across multiple sites.

13. **Gateway ordering rule is critical for dual ZIA+ZPA migrations.** If both ZIA and ZPA are being migrated, ZIA-migrated Gateway network rules (precedence 6000+) could block traffic to Access private apps. The gateway ordering rule at precedence 8000 (`Access Private Apps → Allow`) prevents this. Always enable it in dual migrations.

14. **Orphaned servers produce no routes.** Servers not linked to any server group are detected and warned, but no CIDR routes are created. If these servers host applications, their routes must be added manually.

15. **Disabled ZPA objects are migrated with tags.** Disabled app segments, connector groups, and servers are migrated (not skipped), but tagged `[disabled]` in descriptions. Review whether these should be enabled or removed in CF1.

16. **Unknown SAML attributes are noted but not mapped.** SAML `entryValues` with unrecognized `lhs` keys (not `memberOf`, `groups`, `department`, `email`, `email_domain`) generate a `partial` warning. These attributes are noted in the policy description for manual review.

17. **Access group deduplication threshold is 3.** Reusable Access Groups are only created when 3+ policies share identical condition sets. Policies with 1-2 uses keep inline conditions.

## Related Skills

- **access-app-setup**: Create and configure Access applications. Use for detailed guidance on private destinations, public hostnames, session management, and policy architecture for migrated apps.
- **tunnel-deployment**: Deploy Cloudflare Tunnels. Use to complete the tunnel installation after migration creates tunnel objects — cloudflared install, authentication, and startup.
- **tunnel-routing**: Configure CIDR and hostname routes on tunnels. Use for route troubleshooting and optimization after initial migration.
- **device-posture-setup**: Configure device posture rules and integrations. Use to create the CF posture checks that migrated Access policies reference by name.
- **access-identity**: Set up SCIM provisioning and IdP configuration. **Critical dependency** — must be completed before SAML/SCIM group-based policies can be enforced.
- **gateway-policy-design**: Design Gateway policies. Use to tune the gateway ordering rule and integrate with ZIA-migrated policies.
- **migrate-zscaler-zia**: Migrate Zscaler ZIA internet security policies. **Run in parallel** with ZPA migration for dual-product customers.
- **migrate-palo-alto**: If the customer also has Palo Alto alongside Zscaler, use for PANW-specific migration.
