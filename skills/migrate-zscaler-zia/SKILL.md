---
name: "migrate-zscaler-zia"
description: "Migrate from Zscaler Internet Access (ZIA) to Cloudflare One Gateway. Covers URL filtering → HTTP policies, firewall → Network policies, DLP → DLP profiles, IP/URL groups → Gateway lists, custom URL categories → domain/IP/URL lists, identity (departments/groups/users) → SCIM wirefilter selectors, locations → source IP lists, GRE tunnels → WARP Connector/Magic WAN, and staged rollout."
domain: "migration"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/"
---

# Migrate Zscaler ZIA to Cloudflare One

## Persona

You are a senior Cloudflare One migration architect with deep knowledge of both Zscaler Internet Access (ZIA) and CF1 Gateway. You map ZIA constructs to CF1 equivalents systematically — URL filtering to HTTP policies, firewall rules to network policies, DLP engines to DLP profiles — and flag gaps where ZIA concepts have no direct CF1 equivalent.

## When to Use

- Customer is migrating from Zscaler ZIA to Cloudflare One Gateway
- Customer has a ZIA JSON config export (from ZIA Admin Portal API)
- Customer asks about ZIA-to-CF1 feature parity or policy mapping
- Pre-sales needs a migration assessment for a ZIA displacement deal

## When NOT to Use

- Customer is migrating from ZPA (private access) → use `migrate-zscaler-zpa`
- Customer is migrating from Palo Alto → use `migrate-palo-alto`
- Customer only needs Gateway policy design without migration context → use `gateway-policy-design`
- Greenfield CF1 deployment with no existing ZIA config → use product-specific skills directly

## Assess

### 1. Obtain the ZIA Configuration Export

The migration requires a ZIA JSON export from the ZIA Admin Portal API. Export all relevant keys.

**Required keys in the export:**
- `url_filtering_rules` — URL filtering rules (action, urlCategories, departments, groups, users, state, order)
- `firewall_filtering_rules` — firewall filtering rules (action, protocols, destAddresses, destIpGroups, srcIpGroups, nwApplicationGroups, state, order)
- `ip_destination_groups` — IP destination groups (name, addresses)
- `ip_source_groups` — IP source groups (name, addresses)
- `url_categories` — custom URL categories (configuredName, urls, dbCategorizedUrls, customCategory flag)

**Optional keys:**
- `web_dlp_rules` — DLP rules (action, dlp engine references)
- `dlp_engines` — DLP engines (custom regex patterns)
- `locations` — ZIA locations (name, ipAddresses, country)
- `gre_tunnels` — GRE tunnels (sourceIp)
- `static_ip` — static egress IPs (ipAddress)
- `network_application_groups` — network application groups (partial mapping)
- `network_applications` — individual network applications
- `network_service_groups` — network service groups
- `network_services` — network services (protocol/port definitions)
- `security` — security allowlist (whitelistUrls)
- `security_advanced` — security advanced blocklist (blacklistUrls)
- `users` — individual users referenced in policies
- `department` — departments used for identity scoping
- `groups` — groups used for identity scoping
- `rule_labels` — rule labels for annotation

### 2. Gather Migration Options

| Option | Default | Purpose |
|--------|---------|---------|
| `account_id` | *(required)* | The Cloudflare account ID where Gateway objects will be created. All API calls target this account. Ask the user for this before proceeding. |
| `prefix` | `descaler_` | Prefix for all generated CF1 object names. Enables identification and cleanup. |
| `disabled` | `true` | Generated rules start disabled (audit mode). **Always start disabled** — review before enabling. |
| `scim_configured` | `false` | Is SCIM provisioning active between the customer's IdP and CF1? If `false`, identity references become description annotations only — no wirefilter selectors generated. |

### 3. Assess Scope Complexity

Before designing, quantify:

- **URL filtering rules** — total count, how many reference custom categories, how many have identity scoping
- **Firewall filtering rules** — total count, how many reference IP groups, how many use network application groups (partial mapping)
- **DLP rules + engines** — total count, how many use custom DLP engines with regex (manual migration)
- **IP groups** — destination groups and source groups (each becomes a Gateway IP list)
- **Custom URL categories** — each splits into up to 3 Gateway lists (IP, DOMAIN, URL sublists)
- **Locations** — count locations with IP addresses (source IP lists); locations without IPs need manual config
- **GRE tunnels** — count and note source IPs (migrate to WARP Connector or Magic WAN)
- **Static egress IPs** — require Enterprise entitlement; may fail on import
- **Identity** — count departments, groups, users; determine SCIM readiness
- **Security allow/blocklists** — whitelistUrls and blacklistUrls become Gateway domain/IP lists

### 4. Request Config Files Before Planning

Before proceeding to Design, request all ZIA JSON export files from the customer or PS team. Do not begin rule mapping or object creation until the required files are available — missing files mean rules will be silently unmigratable.

**Request all ZIA JSON export files:**

| File | Purpose |
|------|---------|
| `url_filtering_rules.json` | Primary HTTP policy source — URL filtering rules |
| `ssl_inspection_rules.json` | TLS inspection rules — Do Not Inspect and SSL Block sources |
| `firewall_rules.json` | Firewall filtering rules — Network L4 policy source |
| `url_categories_custom.json` | Custom URL categories — Gateway list source |
| `ip_dest_groups.json` | IP destination groups — IP list source |
| `ip_source_groups.json` | IP source groups — IP list source |
| `network_services.json` | **Critical** — port and protocol definitions; firewall rule service IDs resolve here |
| `network_service_groups.json` | Grouped network service objects |
| `users.json` | Named users referenced in rules (extract emails for identity mapping) |
| `groups.json` | Groups used for identity scoping |
| `department.json` | Departments used for identity scoping |
| `locations.json` | ZIA locations with source IP addresses |
| `gre_tunnels.json` | GRE tunnel source IPs |
| `static_ip.json` | Static egress IPs |
| `web_dlp_rules.json` | DLP rules |
| `dlp_engines.json` | Custom DLP engine regex patterns |
| `security.json` | Security allowlist (`whitelistUrls`) |
| `security_advanced.json` | Security blocklist (`blacklistUrls`) |

**Gate:** Do not proceed to Design until all required files are provided. If a file is missing, document which rule types will be affected and flag those rules for manual review. `network_services.json` is especially critical — without it, firewall rules cannot have port conditions resolved and may be misclassified or dropped entirely.

## Design

### ZIA → CF1 Mapping Overview

ZIA and CF1 Gateway are architecturally similar — both are cloud-based secure web gateways — but differ in how policies, lists, identity, and DLP are structured.

| ZIA Construct | CF1 Construct | Notes |
|---|---|---|
| URL filtering rules | Gateway HTTP policies | Category-based + custom list-based traffic selectors |
| Firewall filtering rules | Gateway Network (L4) policies | IP/port/protocol conditions |
| DLP rules + engines | Gateway DLP policies + DLP profiles | Custom DLP engines need manual regex migration |
| IP destination groups | Gateway IP lists | Direct mapping |
| IP source groups | Gateway IP lists | Used as `net.src.ip` conditions |
| Custom URL categories | Gateway lists (DOMAIN/IP/URL) | Auto-split into typed sublists |
| Security allowlist | Gateway DOMAIN list | From `whitelistUrls` |
| Security blocklist | Gateway DOMAIN list | From `blacklistUrls` |
| Locations (with IPs) | Gateway IP lists (source IPs) | Gateway Locations must be created separately for DNS scoping |
| GRE tunnels | WARP Connector or Magic WAN | Source IPs extracted into IP list; Gateway cannot specify source tunnel |
| Static egress IPs | Dedicated egress IPs | Requires Enterprise entitlement |
| Departments | `identity.groups.name` wirefilter selector | Requires SCIM; NOT Access Groups |
| Groups | `identity.groups.name` wirefilter selector | Requires SCIM; must match IdP group names exactly |
| Users | Manual review | Individual user matching not supported in Gateway wirefilter |

### Transform Pipeline Overview

The ZIA → CF1 migration is a 4-stage pipeline:

```
Stage 1: Lists       — IP groups, custom URL categories, locations, GRE tunnels, security lists → Gateway lists
Stage 2: HTTP Rules  — url_filtering_rules → Gateway HTTP policies (precedence 7000+)
Stage 3: Network     — firewall_filtering_rules → Gateway Network L4 policies (precedence 6000+)
Stage 4: DLP         — web_dlp_rules + dlp_engines → Gateway DLP placeholder policies
```

### Stage 1: List Building

Every ZIA object that contains addresses, URLs, or domains becomes a Gateway list. Lists are created first because policies reference them.

**IP Destination Groups → IP lists:**
Each group's `addresses` array becomes a Gateway IP list named `<prefix><group_name>`.

**IP Source Groups → IP lists:**
Same as destination groups but used in `net.src.ip` conditions.

**Custom URL Categories → split into typed sublists:**
Each custom category (where `customCategory=true`) has its entries classified:
- Entries matching IP/CIDR patterns → `<prefix><catName>-ip` (IP list)
- Entries matching domain patterns → `<prefix><catName>-domain` (DOMAIN list)
- Entries containing path components → `<prefix><catName>-url` (URL list)

The `customCategoryListMap` tracks which lists were created for each category ID, so HTTP policies can reference the correct lists.

**Locations → IP lists:**
Each location's `ipAddresses` become a source IP list: `<prefix>location_<name>`. These are used for source IP scoping in policies.

**GRE Tunnels → IP list:**
All GRE tunnel `sourceIp` values are collected into a single list: `<prefix>gre_tunnel_sources`. Used for `net.src.ip` conditions where tunnel scoping is needed.

**Security Allowlist → DOMAIN + IP lists:**
`security.whitelistUrls` entries split into domain and IP sublists.

**Security Blocklist → DOMAIN + IP lists:**
`security_advanced.blacklistUrls` entries split into domain and IP sublists.

**Entry sanitization:**
- Domains: strip leading `*.` and `.`, strip ports, strip trailing slashes
- URLs: prepend `https://` if no protocol, strip leading dots and trailing slashes
- Deduplication: entries are deduped within each list by type

### Stage 2: HTTP Policies (URL Filtering Rules)

Each ZIA `url_filtering_rules` entry becomes a Gateway HTTP policy.

**Processing order:** Rules are sorted by ZIA `order` field, then assigned CF1 precedence starting at **7000** with step **10** (7000, 7010, 7020, ...).

**Traffic expression building:**
1. Standard URL categories → resolved to CF content/security category IDs via the 90+ entry mapping table
2. Custom URL categories → resolved to `$[list:<listName>]` references for the Gateway lists created in Stage 1
3. Multiple traffic conditions are OR-joined: `(category_expr) or (list_expr)`

**Category mapping:** The transform includes a static mapping table of ~90 Zscaler URL category names to Cloudflare Gateway category IDs. Categories map to either `content_category` or `security_category` IDs in wirefilter expressions:
- Content categories → `any(http.request.uri.content_category[*] in {<ids>})`
- Security categories → `any(http.request.uri.security_category[*] in {<ids>})`
- Unmapped categories → `unmapped` warning emitted

**Action mapping:**

| ZIA Action | CF1 Action | Notes |
|---|---|---|
| `ALLOW` | `allow` | Direct mapping |
| `CAUTION` | `allow` | ZIA "caution" (warn page) → CF allow (no equivalent warn page in Gateway). Also add to description: `[CAUTION_TO_ALLOW: ZIA CAUTION has no CF warn page — confirm allow is acceptable]` |
| `BLOCK` | `block` | Direct mapping |
| `BLOCK_RESET` | `block` | Connection reset variants all → block |
| `BLOCK_DROP` | `block` | Silent drop → block |
| `BLOCK_ICMP` | `block` | ICMP unreachable → block |

**Identity scoping:** If a rule references departments/groups/users, identity conditions are AND-ed with the traffic expression: `(traffic_expr) and (identity_expr)`. See Stage 5 (Identity Resolution) below.

**Rules with no categories are skipped** with a `partial` warning.

### Stage 3: Network Policies (Firewall Filtering Rules)

Each ZIA `firewall_filtering_rules` entry becomes a Gateway Network L4 policy.

**Precedence:** Starting at **6000** with step **10** (6000, 6010, 6020, ...).

**Traffic expression building — multiple condition types AND-joined:**

| ZIA Field | Wirefilter Expression | Notes |
|---|---|---|
| `destAddresses` | `net.dst.ip in {"<ip1>" "<ip2>"}` | Inline IP conditions |
| `destIpGroups` | `net.dst.ip in $[list:<listName>]` | References IP lists from Stage 1 |
| `srcIpGroups` | `net.src.ip in $[list:<listName>]` | Source IP scoping |
| `protocols` | `net.detected_protocol in {"<proto>"} and net.dst.port == <port>` | Protocol → port mapping |
| `destPorts` | `net.dst.port == <port>` or range expressions | Explicit port override |
| `nwApplicationGroups` | `any(net.fqdn.security_category[*] in {188})` **[placeholder]** | Partial mapping — tagged `[manual_update_required]` |

**Protocol mapping table (17 well-known protocols):**

| ZIA Protocol | CF Detected Protocol | Default Port | Notes |
|---|---|---|---|
| `HTTP` | `http` | 80 | |
| `HTTPS` / `SSL` | `tls` | 443 | |
| `SSH` | `ssh` | 22 | |
| `DNS` | `http` | 53 | Also includes `net.protocol == "udp"` |
| `FTP` | — | 21 | No detected_protocol; port-only |
| `IMAP` / `IMAPS` | `imap` | 143 / 993 | |
| `POP3` / `POP3S` | `pop3` | 110 / 995 | |
| `SMTP` / `SMTPS` | `smtp` | 25 / 587 | |
| `MYSQL` | `mysql` | 3306 | |
| `LDAP` / `LDAPS` | `ldap` | 389 / 636 | |
| `NTP` | `ntp` | 123 | Also includes `net.protocol == "udp"` |
| `RSYNC` | `rsync-daemon` | 873 | |

Protocols not in this table emit an `unsupported` warning listing all supported CF detected protocols.

**Network application groups** have no direct CF1 equivalent. They are mapped to a placeholder `security_category {188}` (potentially unwanted software) and tagged `[manual_update_required]`. The user must update traffic expressions to use `any(app.ids[*] in {"..."})` or `net.detected_protocol` selectors.

**Rules with no translatable conditions** are skipped entirely with an `unsupported` warning.

**Network service port resolution from `network_services.json`:**

The `nwServices` array in each firewall rule contains only `{id, name}` — it does NOT contain port ranges. Port definitions are stored separately in `network_services.json`. Always resolve service IDs against this file before classifying a rule as migratable.

Build a lookup map: `service_id → {destTcpPorts, destUdpPorts}` from `network_services.json`. Each port entry has `{start, end?}`:
- TCP exact port: `net.dst.port == <start>`
- TCP port range: `net.dst.port in {<start>..<end>}`
- UDP exact port: `(net.protocol == "udp" and net.dst.port == <start>)`
- UDP port range: `(net.protocol == "udp" and net.dst.port in {<start>..<end>})`
- `TCP_ANY` tag → `net.protocol == "tcp"` (no port restriction)
- `UDP_ANY` tag → `net.protocol == "udp"` (no port restriction)
- `ICMP_ANY` tag → not migratable; document in Not Migrated section

**Firewall rules that must NOT be migrated (document all in the Not Migrated section):**
- Rules that block QUIC for Cloudflare — would block CF infrastructure itself
- Rules using `ICMP_ANY` — no CF Network policy ICMP support
- Rules using `ZSCALER_PROXY_NW_SERVICES` — ZIA-internal construct with no CF equivalent
- Rules where all services are HTTP/HTTPS/DNS-only — already covered by HTTP policies
- UCaaS / O365 One Click managed rules — ZIA-managed, no CF equivalent
- Rules using service types with no `net.detected_protocol` equivalent: NETBIOS, VMware Horizon, H.323, IKE, L2TP, PPTP, SIP, RTSP, MGCP, REAL_MEDIA, GNUTELLA, IRC, GRE_PROTOCOL, ESP_PROTOCOL

### Stage 4: DLP Policies

ZIA DLP is fundamentally different from CF1 DLP. The migration creates **placeholder policies only**.

**DLP engines:** Custom DLP engines with regex patterns emit `manual_review` warnings — ZIA regex patterns must be manually recreated as Cloudflare DLP profiles.

**DLP rules:** Each `web_dlp_rules` entry becomes a disabled HTTP policy with:
- Traffic: `http.request.method == "POST"` (placeholder)
- Description prefixed with `[NEEDS DLP PROFILE]`
- Always disabled regardless of `disabled` option
- Precedence: HTTP range + 500 offset to avoid collision with URL filtering rules

The user must configure actual DLP profile references after creating CF1 DLP profiles.

### Identity Resolution (Cross-cutting)

Identity scoping applies to both HTTP and Network policies. The behavior depends on SCIM configuration:

**When `scim_configured=true`:**
- Departments → `identity.groups.name == "<dept_name>"` wirefilter selectors
- Groups → `identity.groups.name == "<group_name>"` wirefilter selectors
- Users → `manual_review` warning (Gateway cannot match individual users by name; suggest Gateway user lists or `identity.email` selectors)
- Multiple identity conditions are OR-joined: `(identity.groups.name == "A") or (identity.groups.name == "B")`
- Identity expression is AND-ed with traffic: `(traffic) and (identity)`

**When `scim_configured=false`:**
- All identity references noted in policy description as `[ZIA identity: dept "Engineering", group "All Employees"]`
- **No wirefilter selectors generated** — policies match all traffic regardless of identity
- `partial` warning emitted per rule with identity references

**Pre-SCIM email extraction for named users:**
ZIA user display names embed the user's email in format `Name(email@domain.com)`. Even without SCIM, named user identity can be partially preserved:
- Extract the email address from the display name
- Apply selector: `identity.email == "email@domain.com"`
- This selector works without SCIM — it matches the user's login email from the IdP token
- Outcome: *Functional — reduced fidelity* — the rule works for named users but does not scope to groups

For group-only or department-only scoped rules without SCIM:
- Deploy without any identity filter (matches all users — overly permissive)
- Add `[SCIM_REQUIRED: <GroupName>]` to policy description
- Outcome: *Deployed — overly permissive* — do not enable without SCIM

**Critical:** ZIA departments and groups map to `identity.groups.name`, which uses **SCIM-provisioned IdP group membership**. These are NOT Cloudflare Access Groups. Group names must match the IdP groups **exactly** (case-sensitive).

### Precedence Ranges

| Policy Type | Start | Step | Range Example |
|---|---|---|---|
| DNS | 5000 | 10 | 5000, 5010, 5020, ... |
| Network (L4) | 6000 | 10 | 6000, 6010, 6020, ... |
| HTTP | 7000 | 10 | 7000, 7010, 7020, ... |

These ranges are fixed and separated to prevent collision between policy types. ZIA rule order is preserved within each range.

### List Reference Placeholders

Gateway policies reference lists by name using placeholder format: `$[list:<listName>]`. At deploy time (API import, Terraform, or Descaler Direct API), placeholders are resolved to actual list IDs. This means lists must be created before policies.

### Design Decision Matrix

| Scenario | Recommendation | Rationale |
|---|---|---|
| URL filtering rule with standard categories only | Direct category mapping to HTTP policy | 90+ categories mapped; unmapped categories warned |
| URL filtering rule with custom categories | Gateway list references in HTTP policy | Custom categories split into IP/DOMAIN/URL sublists |
| Firewall rule with IP groups | Gateway list references in Network policy | IP lists created in Stage 1 |
| Firewall rule with network application groups | Placeholder + `[manual_update_required]` tag | No direct CF1 mapping; needs manual app.ids or detected_protocol |
| DLP rules with custom engines | Placeholder + manual migration | ZIA regex → CF1 DLP profiles requires manual recreation |
| ZIA CAUTION action | Map to `allow` | CF Gateway has no warn/caution page equivalent |
| Rule with identity scoping, SCIM ready | Generate wirefilter `identity.groups.name` selectors | Direct mapping for departments and groups |
| Rule with identity scoping, no SCIM | Annotate in description only | No selectors generated; re-transform after SCIM setup |
| Rule referencing individual users | Warning + manual review | Gateway wirefilter cannot match by username |
| GRE tunnels | Extract source IPs to Gateway list + warning | CF uses WARP Connector or Magic WAN; no direct GRE tunnel equivalent |
| Static egress IPs | Attempt import + warning | Requires Enterprise entitlement; may fail |
| Dual ZIA + ZPA migration | Run ZIA first (lists/policies), then ZPA (tunnels/Access apps) | ZPA adds gateway ordering rule at 8000 to protect Access private app traffic from Gateway network blocks |

### Migration Readiness Classification

Every source ZIA rule must be assessed for readiness before deployment. Assign one of these five outcome labels to each rule — include the label in the policy description so it is visible in the CF dashboard without cross-referencing documentation.

| Outcome | CF Object Created? | Works Today? | What's Missing |
|---------|-------------------|--------------|----------------|
| **Ready** | Yes | Yes | Nothing — traffic expression, action, and identity (if any) are all correct. Enable when ready. |
| **Functional — reduced fidelity** | Yes | Partially | Named users mapped via `identity.email`. Group-level scope is missing. Needs SCIM provisioning to add `identity.groups[*].name` selectors. |
| **Deployed — overly permissive** | Yes | Yes, but risky | Policy matches all users instead of the intended group. This is a security posture gap. Do not enable without SCIM provisioning. Add `[SCIM_REQUIRED: <GroupName>]` to description. |
| **Needs manual fix** | Yes (placeholder, disabled) | No | Source data incomplete: no URL categories, ZIA-managed built-in lists not exportable, or inline IP list too large. Do not enable without manual remediation. |
| **Not migrated** | No | N/A | No CF equivalent exists, or excluded by design. Documented in the Not Migrated section only. |

**Path to Ready:** Rules currently classified as *Functional — reduced fidelity* or *Deployed — overly permissive* become **Ready** after SCIM provisioning is complete and `identity.groups[*].name` selectors are added.

### Naming Prefix Convention

Every Gateway list and policy created during migration must start with a configured prefix. This enables bulk identification, filtering, and cleanup in the Cloudflare dashboard, and supports safe rollback.

**Default prefix:** `zia_` — configurable per engagement. Set once at the start; apply consistently to every object created.

| Object Type | Naming Pattern | Example |
|------------|---------------|---------|
| IP destination / source groups | `<prefix><group_name>` | `zia_Corp_Servers` |
| Custom category — IP sublist | `<prefix>_custom_<id>_<slug>_ips` | `zia__custom_42_genesys_ips` |
| Custom category — domain sublist | `<prefix>_custom_<id>_<slug>_domains` | `zia__custom_42_genesys_domains` |
| Custom category — URL sublist | `<prefix>_custom_<id>_<slug>_urls` | `zia__custom_42_genesys_urls` |
| HTTP policy (URL filtering) | `<prefix>url_<slug_of_zia_rule_name>` | `zia_url_block_adult_content` |
| HTTP policy (TLS — Do Not Inspect, merged) | `<prefix>ssl_bypass_all` | `zia_ssl_bypass_all` |
| HTTP policy (TLS — Do Not Inspect, individual) | `<prefix>ssl_<slug>` | `zia_ssl_bypass_finance_team` |
| HTTP policy (SSL Block) | `<prefix>ssl_block_<slug>` | `zia_ssl_block_malware` |
| Network L4 policy | `<prefix>fw_<slug_of_zia_rule_name>` | `zia_fw_block_attacker_ips` |

**Why this matters:** In the CF dashboard, filtering by prefix instantly shows all migrated objects. It also makes bulk disabling or deletion safe — you can confidently target only migrated objects without affecting pre-existing policies.

### Custom Category Inline vs List Strategy

ZIA custom categories are collections of URLs, domains, and IPs. They are not migrated as CF objects. Instead, their entries are either embedded inline in the policy traffic expression or placed in a Gateway list, based on entry count.

**Decision rule:**
- **≤5 entries** — embed entries inline directly in the policy's traffic expression. Do not create a Gateway list. Note the ZIA category ID and name in the policy description for traceability back to the ZIA source.
- **>5 entries** — create a named Gateway list using the naming convention above.

**List type must strictly match the entry type:**
- IP list — bare IP addresses and CIDRs
- DOMAIN — hostnames and domain names
- URL — entries that contain a path component

**Entry classification rules:**
- Wildcard entries (`*.domain.com`): strip leading `*.` → DOMAIN list (CF matches apex + all subdomains natively)
- Entries with appended port (`host:8080`): strip port before adding to DOMAIN list
- Entries that look like URLs but are CIDRs (e.g. `https://18.180.0.0/15`): strip `https://` prefix, reclassify as IP list entries
- An entry containing a `/` that is not a CIDR mask is a URL entry, not an IP

### Stage 2b: TLS Inspection Rules (ssl_inspection_rules)

TLS inspection rules from `ssl_inspection_rules.json` become HTTP policies in Cloudflare Gateway — specifically Do Not Inspect policies and HTTP Block policies. Process these after URL filtering rules (Stage 2) and before firewall rules (Stage 3).

**Three ZIA rule types and their CF treatment:**

| ZIA Rule Type | CF Treatment | CF Policy Type |
|---|---|---|
| DECRYPT rules | **Not migrated as objects** — CF inspects by default; these are automatically satisfied when TLS inspection is enabled. Document as: Not Migrated — No CF Equivalent. | — |
| Default SSL rule | **Not migrated** — no CF equivalent. Document as: Not Migrated — No CF Equivalent. | — |
| DO_NOT_DECRYPT — no user or group identity | Merge ALL domain and IP entries from every such rule into ONE policy: `<prefix>ssl_bypass_all`, precedence 3000, enabled. This is the generic bypass policy. | HTTP — Do Not Inspect |
| DO_NOT_DECRYPT — with user or group identity | One individual Do Not Inspect policy per rule. Precedence starting at 3010, step 10. All created disabled. | HTTP — Do Not Inspect |
| SSL BLOCK rules | HTTP block policies. Precedence starting at 7650, step 10. | HTTP — Block |

**Identity handling for DO_NOT_DECRYPT rules with identity:**
- Named users — ZIA user display names follow format `Name(email@domain.com)`. Extract the email and create selector: `identity.email == "email@domain.com"`. Outcome: *Functional — reduced fidelity* (works for named users, missing group scope).
- Group-only scoped rules — no emails available. Deploy without any identity filter. Add `[SCIM_REQUIRED: <GroupName>]` to policy description. Outcome: *Deployed — overly permissive*.

**Stage gate:** After creating all TLS-derived policies, call `GET /accounts/{id}/gateway/rules` and verify that the count of Do Not Inspect policies and SSL Block policies matches the count parsed from `ssl_inspection_rules.json` (excluding DECRYPT rules which intentionally produce no objects). Investigate any mismatch before proceeding to Stage 3.

## Implement

> **Recommended: Connect your IDE or MCP client to the Cloudflare API MCP server**
>
> For the smoothest migration experience, connect your IDE or MCP client to the **Cloudflare API MCP server** before starting implementation. This allows the LLM to call Cloudflare Gateway APIs directly — creating lists, policies, and verifying counts — without manual copy-paste.
>
> The Cloudflare API MCP server provides access to the full Cloudflare API (including all Zero Trust / Gateway endpoints) through two tools: `search()` and `execute()`. Authentication is via **OAuth** — when you first connect, your browser will open and prompt you to authorize access to your Cloudflare account.
>
> **Cloudflare API MCP server URL:** `https://mcp.cloudflare.com/mcp`
>
> **Setup in Windsurf** — add to your MCP config (via Settings > MCP Servers, or `.windsurf/mcp.json`):
> ```json
> {
>   "mcpServers": {
>     "cloudflare-api": {
>       "url": "https://mcp.cloudflare.com/mcp"
>     }
>   }
> }
> ```
>
> **Setup in OpenCode** — add to your `~/.config/opencode/config.json`:
> ```json
> {
>   "$schema": "https://opencode.ai/config.json",
>   "mcp": {
>     "cloudflare-api": {
>       "type": "remote",
>       "url": "https://mcp.cloudflare.com/mcp"
>     }
>   }
> }
> ```
> In OpenCode, OAuth is handled automatically. On first use, run `opencode mcp auth cloudflare-api` to open the browser authorization flow. Tokens are stored securely in `~/.local/share/opencode/mcp-auth.json`.
>
> **References:**
> - Cloudflare API MCP server docs: https://developers.cloudflare.com/agents/model-context-protocol/mcp-servers-for-cloudflare/
> - Cloudflare MCP repository: https://github.com/cloudflare/mcp
> - Cloudflare One documentation: https://developers.cloudflare.com/cloudflare-one/
>
> Once connected, provide your **Cloudflare account ID** when prompted — the LLM will use it in every API call throughout the migration. If your MCP client is not connected, use the Dashboard/API path below instead.

This section provides **three implementation paths** for applying the migration design. Choose based on tooling access:

| Path | When to Use | Prerequisites |
|------|------------|---------------|
| **Automated (Descaler)** | Full migration with many rules. Handles the entire 4-stage pipeline in one pass. | Access to a Descaler deployment |
| **MCP Tools** | Incremental migration, or when Descaler is unavailable. You drive each stage using CF1 MCP tools. | MCP server connected with Gateway app |
| **Dashboard / API** | No tooling available, or for manual review of individual policies. | CF dashboard access or API token with `Zero Trust: Edit` scope |

All three paths use the **same migration options** from Assess § 2. When using MCP or Dashboard/API, apply options manually at each step — prepend `prefix` to every name, set rules disabled when `disabled=true`, include identity selectors only when `scim_configured=true`.

### Automated Migration (Descaler)

> **Descaler** is a Cloudflare Workers app for automated migration transforms. Deploy your own instance with `npm run build && npm run deploy`.

1. Open the Descaler UI in your browser
2. Select **"Zscaler ZIA"** as the source vendor
3. Upload the ZIA JSON export
4. Configure migration options in the UI:
   - **Prefix** — object name prefix (default: `descaler_`)
   - **Start disabled** — toggle on (recommended)
   - **SCIM configured** — toggle on only if SCIM is live between the customer's IdP and CF1
5. Click **Transform** → review output and all warnings
6. Choose export format:
   - **JSON** — raw `CloudflareConfig` for inspection or custom tooling
   - **Terraform** — HELIX-compatible `.tf` file using `var.account_id`, precedence ranges preserved
   - **Python script** — standalone `deploy.py` with dry-run support, reads credentials from env vars
   - **Direct API** — browser-based import via CF API proxy with progress tracking (creates lists first, resolves `$[list:]` placeholders, then creates policies)
7. **Review all warnings** before deploying — see Validate § 1

### With MCP Tools (Code Mode)

Execute in dependency order: Lists → HTTP policies → Network policies → DLP policies.

**Step 1: Create Gateway Lists (from IP groups, custom categories, locations, etc.)**
```
→ search: "create gateway list" — find POST endpoint and request body schema
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/lists`,
    body: {
      name: "<prefix><group_name>",
      type: "IP",
      description: "Migrated from ZIA IP destination group: <name>",
      items: [
        { value: "10.0.0.0/8" },
        { value: "192.168.1.0/24" }
      ]
    }
  })
```

For custom URL categories that split into multiple types, create one list per type:
```
"<prefix><catName>-ip"     → type: "IP"
"<prefix><catName>-domain" → type: "DOMAIN"
"<prefix><catName>-url"    → type: "URL"
```

**Step 2: Create Gateway HTTP Policies (from URL filtering rules)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix><rule_name>",
      description: "Migrated from ZIA URL filtering rule",
      action: "block",
      traffic: "any(http.request.uri.content_category[*] in {67 133})",
      filters: ["http"],
      precedence: 7000,
      enabled: false
    }
  })
```

For rules referencing custom categories, replace `$[list:<listName>]` with the actual list ID:
```
"traffic": "any(http.request.domains[*] in $<list_id>)"
```

For rules with identity scoping (SCIM configured):
```
"traffic": "(any(http.request.uri.content_category[*] in {67})) and (identity.groups.name == \"Engineering\")"
```

**Step 3: Create Gateway Network Policies (from firewall rules)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix><rule_name>",
      description: "Migrated from ZIA firewall rule",
      action: "block",
      traffic: "net.dst.ip in $<list_id> and net.dst.port == 443",
      filters: ["l4"],
      precedence: 6000,
      enabled: false
    }
  })
```

For rules tagged `[manual_update_required]` (network application groups):
- Create the policy with the placeholder expression
- Review and update the traffic expression to use `any(app.ids[*] in {"..."})` or `net.detected_protocol`

**Step 4: Create DLP Placeholder Policies**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix><dlp_rule_name>",
      description: "[NEEDS DLP PROFILE] Migrated from ZIA DLP rule",
      action: "block",
      traffic: "http.request.method == \"POST\"",
      filters: ["http"],
      precedence: 7500,
      enabled: false
    }
  })
```
After creating CF1 DLP profiles, update these policies with actual DLP profile references.

### Stage Gate Verification (MCP Path)

After each stage, verify object counts via API before proceeding. **The principle:** the count of CF objects created must match the count of source rules parsed from the ZIA JSON files. A mismatch means something was silently skipped — stop and investigate before continuing.

| Stage | Verification Call | What to Check |
|-------|-----------------|---------------|
| After Stage 1 (Lists) | `GET /accounts/{id}/gateway/lists` | Count must match: IP destination groups + IP source groups + custom category sublists (each category produces up to 3 lists) + location lists + security allow/block lists |
| After Stage 2 (HTTP URL policies) | `GET /accounts/{id}/gateway/rules` (filter: `http`) | Every `url_filtering_rules` entry must have a corresponding HTTP policy — including placeholders for rules with no categories |
| After Stage 2b (TLS policies) | `GET /accounts/{id}/gateway/rules` (filter: `http`) | 1 merged Do Not Inspect policy + one per identity-scoped DO_NOT_DECRYPT rule + one per SSL BLOCK rule. DECRYPT rules intentionally produce no objects. |
| After Stage 3 (Network L4 policies) | `GET /accounts/{id}/gateway/rules` (filter: `l4`) | Count must match the number of firewall rules eligible for migration (total source rules minus those on the explicit not-migrate list) |
| Final cross-check | Both endpoints | Every source ZIA rule across all rule types must map to either a CF object or a Not Migrated entry. No gaps permitted. |

If any count does not match: identify which source rule has no corresponding CF object, determine whether it should be a placeholder or a Not Migrated entry, and resolve before proceeding to the next stage.

### Without MCP Tools (Dashboard/API)

**Gateway Lists:**
1. Dashboard: Zero Trust → Gateway → Lists → Create a list
2. Choose type (IP, DOMAIN, URL), add items from ZIA export
3. Name with prefix for identification

**API equivalent:**
```bash
# Create a Gateway IP list
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/lists" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "descaler_Corp_Servers",
    "type": "IP",
    "description": "Migrated from ZIA IP destination group",
    "items": [{"value": "10.0.0.0/8"}, {"value": "172.16.0.0/12"}]
  }'

# Create a Gateway HTTP policy
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "descaler_Block_Adult",
    "action": "block",
    "traffic": "any(http.request.uri.content_category[*] in {67 133})",
    "filters": ["http"],
    "precedence": 7000,
    "enabled": false
  }'
```

**Gateway HTTP Policies:**
1. Dashboard: Zero Trust → Gateway → Firewall Policies → HTTP
2. Add policy: set traffic selector using content/security categories or list references
3. Set action (allow/block), precedence, and start disabled

**Gateway Network Policies:**
1. Dashboard: Zero Trust → Gateway → Firewall Policies → Network
2. Add policy: set traffic selector with IP, port, protocol conditions
3. For rules with network app groups: add placeholder, tag for manual update

**DLP Policies:**
1. Dashboard: Zero Trust → Gateway → DLP Profiles → Create profile
2. Manually recreate ZIA DLP engine regex patterns as CF1 DLP entries
3. Create Gateway HTTP policy referencing the DLP profile

**Gateway Locations (for DNS scoping):**
1. Dashboard: Zero Trust → Gateway → Locations → Add a location
2. Configure source IPs from ZIA location data
3. This is separate from the IP lists — locations enable DNS policy scoping

## Validate

### 1. Review Warnings

Every migration produces warnings. Review them **before enabling any policies**:

- **`unmapped`** — ZIA URL categories with no Cloudflare mapping. The category name is in the warning message. Check if a new CF category has been added, or create a custom list to match the content.
- **`partial`** — Rules skipped due to empty categories or empty traffic expressions, locations without IPs, network services flattened, identity references without SCIM, GRE tunnel source IPs extracted, static egress IPs needing entitlements, departments/groups mapped to `identity.groups.name`.
- **`unsupported`** — Firewall rules with no translatable conditions (skipped entirely), protocols with no CF `detected_protocol` equivalent.
- **`manual_review`** — Network application groups needing manual `app.ids` or `detected_protocol` updates, individual user references in policies, custom DLP engines needing regex recreation, network applications needing manual mapping.

### 2. Verify Object Counts

Check that the migration produced the expected number of objects:

```
→ execute: GET `/accounts/${accountId}/gateway/lists`    — verify Gateway lists (IP groups + custom categories + locations + security lists)
→ execute: GET `/accounts/${accountId}/gateway/rules`    — verify Gateway rules by filter type:
                                filters: ["http"]  — URL filtering rules + DLP placeholders
                                filters: ["l4"]    — firewall filtering rules
```

Cross-reference with pipeline output summary: lists created, HTTP rules, network rules, DLP rules, warnings count.

### 3. Verify Category Mappings

For HTTP policies using standard URL categories:
1. Pick 3-5 representative rules with category-based traffic expressions
2. Verify the CF category IDs match the intended ZIA categories
3. Test with known URLs that should match those categories
4. Check for `unmapped` warnings — these categories need manual attention

### 4. Test Policy Behavior

Enable policies one at a time in a test environment:
1. **HTTP allow rules first** — verify allowed traffic passes through
2. **HTTP block rules** — verify blocked categories/domains are blocked
3. **Network allow rules** — verify IP/port conditions match expected traffic
4. **Network block rules** — verify firewall blocks work as intended
5. **Identity-scoped rules** (if SCIM configured) — test with users in referenced groups
6. **DLP rules** — after manual DLP profile configuration, test with sample sensitive data

### 5. Edge Case Checks

- Rules tagged `[manual_update_required]` — verify network app group traffic expressions have been updated
- Rules tagged `[needs_identity]` — verify these are placeholder until user list or identity configuration
- DLP rules prefixed `[NEEDS DLP PROFILE]` — verify DLP profiles have been created and referenced
- Custom category lists — verify IP/DOMAIN/URL sublists contain expected entries
- List reference placeholders `$[list:]` — verify all have been resolved to actual list IDs

### 6. Not Migrated Cross-Check

Before declaring the migration complete, verify that every source ZIA rule is accounted for. This is the completeness gate.

**The discipline:** For every rule in `url_filtering_rules.json`, `ssl_inspection_rules.json`, and `firewall_filtering_rules.json`, the rule must end up in exactly one of two places:
1. A CF Gateway object (list or policy) exists that corresponds to it, OR
2. An explicit Not Migrated entry documents it

**No rule may be silently dropped.** A warning log alone is not sufficient — every unsupported or skipped rule must produce a row in the Not Migrated table. The migration is complete only when the CF object list and the Not Migrated table together account for 100% of source rules.

**Not Migrated table — required columns:**

| Rule Name | Rule Type | Reason | CF Equivalent | Security Posture Impact |
|-----------|-----------|--------|--------------|------------------------|
| *(example) Block QUIC for CF* | Firewall L4 | Would block CF infrastructure | Not Created | None — correct to exclude |
| *(example) Decrypt Finance Traffic* | TLS DECRYPT | CF inspects by default | Not Created — auto-satisfied | None |

## Operate

### Staged Rollout

**Phase 1: Audit mode (disabled=true)**
1. Deploy all lists and policies with `disabled=true`
2. Review all warnings and tagged rules
3. Update `[manual_update_required]` rules with correct traffic expressions
4. Configure DLP profiles for `[NEEDS DLP PROFILE]` rules

**Phase 2: Enable in waves**
1. Enable HTTP allow rules first (least risk)
2. Enable HTTP block rules for non-controversial categories (malware, phishing)
3. Enable network rules, starting with well-understood IP-based blocks
4. Enable identity-scoped rules after verifying SCIM group matching
5. Enable DLP rules last (highest impact, needs most testing)

**Phase 3: Decommission ZIA**
1. Verify all traffic flows through CF1 with expected behavior
2. Monitor for missed rules or unexpected blocks
3. Gradually shift user traffic from ZIA to CF1 (split tunnel or DNS cutover)
4. Keep ZIA active for rollback until confidence is established

### Monitoring

- **Gateway activity logs**: Monitor HTTP and Network policy matches (filter by prefix)
- **Gateway analytics**: Compare block/allow rates against ZIA baseline
- **Identity logs**: Verify SCIM group-based policies are matching expected users
- **DLP logs**: Monitor DLP payload matches after enabling DLP rules
- **List usage**: Verify Gateway lists are being referenced by active policies

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| HTTP policy not matching expected traffic | Wrong category ID mapping | Check `unmapped` warnings; verify CF category IDs in traffic expression |
| Network policy not blocking | Placeholder traffic expression | Check for `[manual_update_required]` tag; update app group expression |
| Identity policy matching all users | SCIM not configured | Enable SCIM, re-transform, or manually add `identity.groups.name` selectors |
| Identity policy matching wrong users | Group name mismatch | Verify IdP group names match ZIA department/group names exactly (case-sensitive) |
| DLP policy not triggering | Missing DLP profile reference | Replace `http.request.method == "POST"` placeholder with actual DLP profile conditions |
| Custom category list not matching | Entry sanitization issue | Check if wildcard or port was stripped; verify entries in Gateway list |
| GRE tunnel traffic not routing | GRE → WARP/Magic WAN not configured | Deploy WARP Connector or Magic WAN tunnel; update source IP conditions |
| Static egress IP failing | Missing Enterprise entitlement | Confirm account has dedicated egress entitlement; contact account team |
| Precedence collision between policy types | Policies in wrong range | Verify HTTP policies at 7000+, network at 6000+, DLP at 7500+ |

### When to Revisit

- **Post-SCIM setup**: Re-transform with `scim_configured=true` to generate `identity.groups.name` wirefilter selectors for all identity-scoped rules
- **Post-DLP profile creation**: Update DLP placeholder policies with actual DLP profile conditions
- **Network app group updates**: After identifying CF app IDs for ZIA network application groups, update `[manual_update_required]` rules
- **New ZIA rules added during migration**: Re-export ZIA config and run incremental transform
- **CF category updates**: If Cloudflare adds new URL categories, check previously `unmapped` categories

## Gotchas

1. **Network application groups are PARTIALLY mapped.** ZIA network application groups have no direct CF1 equivalent. They are mapped to a placeholder `security_category {188}` and tagged `[manual_update_required]`. You MUST review these rules and update traffic expressions to use `any(app.ids[*] in {"..."})` or `net.detected_protocol` selectors.

2. **Network services are flattened.** ZIA allows grouped service objects (e.g., "Web Services" = HTTP+HTTPS). CF1 has no grouped service objects — each protocol/port becomes an individual condition in the wirefilter expression. This is a structural difference, not a bug.

3. **ZIA locations with IPs → Gateway lists, NOT Gateway Locations.** IP addresses from ZIA locations become Gateway IP lists for source IP conditions in policies. However, Cloudflare Gateway Locations (for DNS policy scoping) must be created separately via `/gateway/locations` API. The transform does NOT create Gateway Locations.

4. **GRE tunnels → WARP Connector or Magic WAN.** CF1 does not have GRE tunnel support. ZIA GRE tunnel source IPs are extracted into a Gateway IP list for `net.src.ip` conditions. The actual tunnel migration requires deploying WARP Connector or Magic WAN tunnels as a separate workstream. Gateway policies cannot specify source tunnel directly.

5. **Static egress IPs require Enterprise entitlement.** The transform will attempt to configure dedicated egress IP policies, but the API will fail if the account lacks the Enterprise entitlement. Verify entitlements before import.

6. **Without SCIM: identity references are DESCRIPTION-ONLY.** When `scim_configured=false`, ZIA department/group/user references become annotations in policy descriptions (e.g., `[ZIA identity: dept "Engineering"]`). No wirefilter selectors are generated. Policies match ALL traffic, not just the intended users. This is by design — generating broken identity selectors would be worse.

7. **ZIA departments/groups map to `identity.groups.name`, NOT Access Groups.** The `identity.groups.name` selector uses SCIM-provisioned IdP group membership. Cloudflare Access Groups are a different concept and CANNOT be used in Gateway policies. If ZIA departments do not exist as IdP groups, create Gateway user lists instead.

8. **Group names must match IdP groups EXACTLY (case-sensitive).** If ZIA has department "Engineering" but the IdP has group "engineering" (lowercase), the wirefilter selector will not match. Verify group names after SCIM setup.

9. **Individual user references generate manual review warnings.** Gateway wirefilter cannot match individual users by name. ZIA rules referencing specific users are tagged for manual review. Consider creating a Gateway user list with their emails, or use `identity.email` selectors if SCIM provides email claims.

10. **ZIA `CAUTION` action maps to `allow`.** ZIA's "caution" action shows a warning page before allowing access. CF1 Gateway has no equivalent warn/caution page. These rules are mapped to `allow` — if blocking is preferred, change the action manually.

11. **Custom URL categories split into up to 3 sublists.** A single ZIA custom category with mixed entries (IPs, domains, URLs) produces up to 3 Gateway lists (`-ip`, `-domain`, `-url`). HTTP policies reference all sublists from that category. Be aware of this multiplication when counting expected lists.

12. **DLP migration is placeholder-only.** ZIA DLP engines use proprietary regex patterns that cannot be automatically converted to CF1 DLP profiles. Every DLP rule produces a disabled placeholder policy with `http.request.method == "POST"` as the traffic expression. You MUST manually create CF1 DLP profiles and update these policies.

13. **Custom DLP engines need manual regex recreation.** ZIA custom DLP engines with regex expressions are flagged with `manual_review` warnings. Each regex must be manually translated to a Cloudflare DLP custom entry or dataset.

14. **List reference placeholders must be resolved before deployment.** Policies contain `$[list:<listName>]` placeholder references. The Descaler Direct API and Terraform export handle this automatically. For manual or MCP deployment, you must create lists first, obtain their IDs, and substitute the placeholders.

15. **Protocols without CF `detected_protocol` support are dropped.** Only 17 well-known protocols have CF mappings. Unsupported protocols (e.g., SCTP, custom protocols) emit `unsupported` warnings and generate no traffic conditions. If the entire rule relies on unsupported protocols, it is skipped.

16. **DNS rules are not generated from URL filtering rules.** The current transform produces HTTP policies from URL filtering rules but does not generate DNS policies. DNS policy creation (using `dns.content_category` and `dns.security_category` selectors) is a potential future enhancement. Create DNS policies manually if needed.

17. **Deploy with `disabled=true` first — ALWAYS.** Even if ZIA rules were enabled, deploy to CF1 with `disabled=true` and review all policies, warnings, and tagged rules before enabling. This is the single most important operational practice for migration safety.

18. **Rule labels are preserved in descriptions.** ZIA rule labels are resolved and appended to policy descriptions as `{labels: "Migrated", "Security"}`. They are informational only and do not affect policy behavior.

## Related Skills

- **gateway-policy-design**: Design Gateway policies after migration. Use for custom additions, DNS policies, and tuning migrated rules.
- **gateway-dlp**: DLP policy design and profile creation. **Critical dependency** — must be completed before DLP placeholder policies can be activated.
- **gateway-tls-inspection**: Configure TLS inspection for HTTP policies. Required for HTTP policies to inspect encrypted traffic.
- **gateway-dns-security**: DNS security policy design. Use if ZIA DNS filtering needs to be replicated (not auto-migrated).
- **access-identity**: Set up SCIM provisioning and IdP configuration. **Critical dependency** — must be completed before identity-based policies can be enforced.
- **migrate-zscaler-zpa**: Migrate Zscaler ZPA private access policies. **Run in parallel** with ZIA migration for dual-product customers. ZPA adds a gateway ordering rule at precedence 8000 to protect Access private app traffic.
- **migrate-palo-alto**: If the customer also has Palo Alto alongside Zscaler, use for PANW-specific migration.

## Report

At the conclusion of the migration, generate two output documents. Save to the agreed output location (e.g. `~/Downloads/` or a shared drive folder).

### Document 1: `<PREFIX>_ZIA_Migration_Guide.docx` (PS Engineer Reference)

Internal reference for the Cloudflare PS engineer and technical team.

| Section | Content |
|---------|---------|
| 1. Purpose and Scope | Migration objectives, source environment summary, CF account details |
| 2. Migration Effort Reality | Scorecard (% Automated / Needs SCIM or Fix / Not Migrated); outcome label legend table; visual bar charts per rule type showing how many rules became which CF policy type and with which outcome; 'What AI Automates vs What Requires Human Judgment' table |
| 3. Design Decisions | Custom category strategy (inline ≤5 / list >5), identity pre-SCIM approach, TLS rule classification, firewall rule classification rationale, CAUTION→ALLOW mapping, category substitutions |
| 4. CF Objects Created | Gateway lists table; HTTP URL policies with precedence ranges; TLS Do Not Inspect policies; SSL Block policies; Network L4 policies — all with counts |
| 5. Not Migrated / Out of Scope | Table: Item \| Count \| Reason \| Security Posture Impact |
| 6. Manual Actions Required | Table: Priority \| Item \| Why Manual \| Steps Required \| Status |
| 7. Verification Guide | API verification calls, dashboard paths, expected counts, staged enablement order |
| 8. Wirefilter Expression Reference | HTTP / Do Not Inspect / Network L4 field names with examples |
| 9. The Optimised Migration Prompt | The structured prompt used for this engagement in a shaded box, with a placeholder substitution table |
| 10. Recreating This Migration | Step-by-step guide for future engagements using the same methodology |

### Document 2: `<PREFIX>_ZIA_Migration_Report_Customer.docx` (Customer Executive Report)

Customer-facing report for stakeholder review and sign-off.

| Section | Content |
|---------|---------|
| 1. Executive Summary | Migration-at-a-glance table (Object Type \| ZIA Source Count \| CF Created \| Status); key decisions made; important notes for customer review |
| 2. Automation Coverage | Scorecard (% automated / needs action / not migrated); visual bar charts per rule type with plain-English notes; SCIM callout as the highest-leverage next action |
| 3. Design Decisions | 3.1 Custom URL Category Handling; 3.2 Identity & Group Controls (ZIA scope \| current CF \| after SCIM); 3.3 TLS Inspection Architecture Change (ZIA model \| CF model \| approach taken); 3.4 ZIA Predefined Category → CF Mapping table |
| 4. Migration Status — All Rules | 4.1 URL Filtering Rules status (one row per rule); 4.2 TLS Inspection Rules status; 4.3 Firewall Rules status. Every table must include the CF Policy Name column using exact names created in Cloudflare. |
| 5. Not Migrated / Out of Scope | Table: Item \| Count \| Reason \| Impact on Security Posture |
| 6. Manual Actions Required | Table: Priority \| Item \| Why Manual \| Steps Required \| Status |
| 7. Staged Enablement Roadmap | Table: Phase \| Timeline \| Policies to Enable \| Risk \| Gate Criteria |
| 8. Verification Guide | Object count verification (dashboard paths + expected counts); post-enablement monitoring checklist |

**Migration status table columns (all rule-type tables in Document 2):**
`# | ZIA Rule Name | ZIA Action | CF Policy Name | CF Policy Type | Status | Notes`

- **CF Policy Name**: exact name created in Cloudflare (e.g. `zia_url_block_adult_content`); use "Not Created" for not-migrated rules
- **CF Policy Type**: `HTTP Policy` | `HTTP — Do Not Inspect` | `HTTP — Block` | `Network L4` | `Not Created`
- **Status**: `Verified` | `Partially Migrated` | `Not Migrated — No CF Equivalent` | `Not Migrated — Config Not Provided`
