---
name: "migrate-palo-alto"
description: "Migrate from Palo Alto Networks (PAN-OS, Prisma Access, GlobalProtect) to Cloudflare One. Covers security rule classification into Gateway/Access/Posture, address/service object resolution, App-ID mapping, HIP profile → device posture, LDAP identity → SCIM groups, and GlobalProtect → CF1 Client transition."
domain: "migration"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/"
---

# Migrate Palo Alto Networks to Cloudflare One

## Persona

You are a senior Cloudflare One migration architect with deep knowledge of both Palo Alto Networks (PAN-OS, Panorama, Prisma Access, GlobalProtect) and CF1. You understand that a single PANW security rule can decompose into multiple CF1 constructs — Gateway Network rules, Gateway HTTP/DNS rules, Access applications, Access policies, Gateway lists, and device posture rules — and you navigate these mappings with precision.

## When to Use

- Customer is migrating from Palo Alto (PAN-OS, Panorama, Prisma Access, GlobalProtect) to CF1
- Customer has a PANW config export — PAN-OS `set` commands, CSV/XLSX security rules, address objects/groups, service objects/groups, HIP profiles
- Customer asks about PANW-to-CF1 feature mapping or competitive assessment
- Pre-sales needs a migration scope estimate (rule count, identity complexity, posture requirements)

## When NOT to Use

- Customer is migrating from Zscaler → use `migrate-zscaler-zia` (internet security) or `migrate-zscaler-zpa` (private access)
- Customer only needs Gateway policy design without migration context → use `gateway-policy-design`
- Customer only needs Access app setup without migration context → use `access-app-setup`
- Greenfield CF1 deployment with no existing PANW config → use product-specific skills directly

## Assess

### 1. Obtain the PANW Configuration Export

The migration requires structured data from PAN-OS or Panorama. Two input formats are supported:

**PAN-OS `set` commands** (preferred — most complete):
```
set device-group CORP-PROD pre-rulebase security rules ALLOW_WEB source [ 10.0.0.0/8 ]
set device-group CORP-PROD pre-rulebase security rules ALLOW_WEB destination [ WebServers ]
set device-group CORP-PROD pre-rulebase security rules ALLOW_WEB application [ web-browsing ssl ]
set device-group CORP-PROD pre-rulebase security rules ALLOW_WEB service application-default
set device-group CORP-PROD pre-rulebase security rules ALLOW_WEB action allow
set device-group CORP-PROD address WebServer1 ip-netmask 10.226.1.177/32
set device-group CORP-PROD address-group WebServers static [ WebServer1 WebServer2 ]
set device-group CORP-PROD service tcp-389 protocol tcp port 389
set device-group CORP-PROD profiles hip-profiles CORP_POSTURE match "host-info" has "disk-encryption"
```

**Tabular CSV/XLSX** (from Panorama export or Expedition):
- Security rules sheet: Name, Source, Destination, Application, Service, Action, Source User, HIP Profiles, Category
- Address objects sheet: Name, Type (ip-netmask/ip-range/fqdn), Value
- Service objects sheet: Name, Protocol, Port
- HIP profiles sheet: Name, Match Expression

### 2. Gather Migration Options

These options control the transform behavior:

| Option | Default | Purpose |
|--------|---------|---------|
| `prefix` | `descaler_` | Prefix for all generated CF1 object names. Enables easy identification and cleanup. |
| `disabled` | `true` | Whether generated rules start disabled. **Always start disabled** — review before enabling. |
| `scim_configured` | `false` | Is SCIM provisioning active between the customer's IdP and CF1? If `false`, LDAP group references become description annotations only. |
| `posture_configured` | `false` | Should HIP profiles be transformed to device posture rules? If `false`, HIP profiles are noted but not converted. |
| `session_duration` | `24h` | Default Access application session duration. |
| `default_email_domain` | `""` | Email domain for constructing identity conditions (e.g., `corp.example.com`). |
| `ignore_appid_any` | `true` | Treat `application=any` as "no application restriction" (skip App-ID mapping). |
| `fallback_to_port` | `true` | When an App-ID can't be mapped to a CF detected protocol, fall back to port-only matching. |

### 3. Assess Scope Complexity

Before designing, quantify:

- **Total security rules** — how many, how many disabled (will be skipped), how many identity-aware
- **Address objects/groups** — recursive groups increase resolution complexity
- **Service objects/groups** — custom services vs. well-known (SSH, HTTPS, etc.)
- **HIP profiles** — which checks (AV, disk encryption, firewall, EDR, domain join)?
- **Identity references** — LDAP DNs in `source-user` fields. Requires SCIM to become enforceable.
- **App-IDs** — which PANW applications are used? Check if they have CF protocol equivalents.
- **URL categories** — rules using PANW URL categories need manual mapping to CF security categories.
- **Schedule-based rules** — not supported in CF1, will be skipped.

## Design

### Transform Pipeline Overview

The PANW → CF1 migration is a 3-stage pipeline:

```
Stage 1: Resolve Objects
  address_objects + address_groups → address_map (name → [IP/CIDR/FQDN])
  service_objects + service_groups → service_map (name → [protocol/port])

Stage 2: Classify & Transform Security Rules
  Each rule → classify → one of 4 buckets:
    identity_access   → Access Application + Access Policy
    network_allow      → Gateway Network L4 rule (+ DNS rule for FQDNs, + HTTP rule for URL categories)
    network_deny       → Gateway Network L4 block rule
    unsupported        → Warning only (disabled rules, schedule-based rules)

Stage 3: Map HIP Profiles → Device Posture Rules
  HIP match expressions → CF posture rule types (disk_encryption, firewall, os_version, etc.)
```

### Stage 1: Object Resolution

Address and service objects are resolved recursively. Groups can contain other groups.

**Address resolution:**
- `ip-netmask` → classified as `ip` (single host /32) or `cidr`
- `ip-range` → treated as CIDR-like
- `fqdn` → maps to hostname destinations (Access) or domain lists (Gateway DNS)
- Circular group references are detected and warned
- Unresolvable references (not in any object/group) generate `unmapped` warnings

**Service resolution:**
- Named service objects → protocol + port pairs
- `application-default` → look up App-ID default ports from Applipedia
- Well-known inline patterns recognized: `TCP-22`, `service-https`, `service-443-TCP`, `UDP-53`
- Built-in well-known services: http(80), https(443), ssh(22), dns(53), rdp(3389), smtp(25), ldap(389), ldaps(636)

### Stage 2: Security Rule Classification

This is the core of the migration. Each PANW security rule is classified into exactly one bucket:

| Classification | Trigger Conditions | CF1 Output |
|---|---|---|
| **identity_access** | `source_user` != `any` (has LDAP DN references) | Access Application (private, deduped by destination+port) + Access Policy (allow/deny) |
| **network_allow** | No identity + (URL categories present, OR destinations are external) | Gateway Network L4 allow + optional DNS rule for FQDNs + optional HTTP rule for URL categories |
| **network_deny** | Action is deny/drop/reset-* + no identity | Gateway Network L4 block |
| **unsupported** | Rule is disabled, OR has a schedule | Skipped with warning |

**Classification decision tree:**
1. Disabled rule → `unsupported`
2. Schedule-based rule → `unsupported`
3. Action is deny/drop/reset-* → `identity_access` if has identity, else `network_deny`
4. Has URL categories → `network_allow` (internet-bound SWG)
5. Has identity + internal destinations → `identity_access`
6. Has identity (any destination) → `identity_access`
7. Internal or external network-only → `network_allow`
8. Fallback → `network_allow`

### Identity Handling (LDAP → SCIM)

PANW `source-user` fields contain LDAP Distinguished Names:
```
CN=VPN_OPS_IT OU=Groups DC=sso DC=indeed DC=net
CN=VPN_DevOps,OU=Groups,DC=sso,DC=indeed,DC=net
```

Both space-delimited (PAN-OS display format) and comma-delimited (standard LDAP) are parsed. The `CN` value is extracted as the group name.

| SCIM Status | Access Policy Behavior | Gateway Policy Behavior |
|---|---|---|
| **SCIM ON** | `include` conditions use `group` type with CN as the SCIM group name | Wirefilter: `identity.groups.name == "VPN_OPS_IT"` |
| **SCIM OFF** | `include` is set to `everyone`; groups noted in description: `[PANW identity: groups "VPN_OPS_IT" (domain: sso.indeed.net)]` | Groups noted in rule description only |

**Reusable Access Groups:** When a SCIM group name appears in 2+ policies, a reusable Access Group is automatically created to reduce duplication.

### App-ID Mapping (Applipedia)

PANW App-IDs are mapped to CF1 constructs via a static lookup table (~70 entries). Key mappings:

| PANW App-ID | CF `net.detected_protocol` | Default Ports |
|---|---|---|
| `ssh` | `ssh` | TCP/22 |
| `ssl` | `tls` | TCP/443 |
| `web-browsing` | `http` | TCP/80 |
| `dns` | `dns` | UDP/53, TCP/53 |
| `ms-rdp` | `rdp` | TCP/3389 |
| `mysql` | `mysql` | TCP/3306 |
| `smtp` | `smtp` | TCP/25, TCP/587 |
| `ldap` | `ldap` | TCP/389 |

When an App-ID has a `cf_detected_protocol`, the generated wirefilter includes `net.detected_protocol in {"ssh"}`. When it doesn't (e.g., `ms-teams`, `zoom`, `slack`), only port-based matching is used.

**Unknown App-IDs** generate an `unmapped` warning and fall back to port-only matching.

### HIP Profile → Device Posture Mapping

HIP match expressions are parsed for known check types:

| HIP Check | CF Posture Type | Notes |
|---|---|---|
| `anti-virus` / `antivirus` | `application` | Map to your specific AV vendor in CF |
| `disk-encryption` | `disk_encryption` | BitLocker (Win), FileVault (Mac), LUKS (Linux) |
| `firewall` | `firewall` | Host firewall enabled |
| `domain` / `domain-joined` | `domain_joined` | Windows only |
| `os-version` / `minimum-version` | `os_version` | Configure min version in CF |
| `crowdstrike` | `crowdstrike_s2s` | Requires CF s2s integration setup |
| `sentinelone` / `sentinel` | `sentinelone_s2s` | Requires CF s2s integration setup |
| `tanium` | `tanium_s2s` | Requires CF s2s integration setup |
| `workspace` / `ws1` | `workspace_one` | Requires CF integration setup |
| Unrecognized | `file` (placeholder) | Manual review required |

Platform detection: If the HIP expression specifies `"host-info" is "Windows"` or `"Mac"`, only those platforms are included. If no platform is specified, defaults to Windows + Mac + Linux. Domain join checks are filtered to Windows only.

### Precedence Ranges

Generated rules use reserved precedence ranges to avoid collision with existing policies:

| Type | Starting Precedence | Increment |
|---|---|---|
| Gateway DNS rules | 5000 | +10 per rule |
| Gateway Network L4 rules | 6000 | +10 per rule |
| Gateway HTTP rules | 7000 | +10 per rule |

### Design Decision Matrix

| Scenario | Recommendation | Rationale |
|---|---|---|
| PANW rule with identity + internal destinations | Access Application (private) + Access Policy | Identity-aware access to private resources → Access is the right construct |
| PANW rule with URL categories | Gateway HTTP rule | URL category filtering is a Gateway SWG function |
| PANW rule with destination FQDNs | Gateway DNS rule (for the domain list) + Gateway Network L4 rule (for IPs) | FQDNs need DNS-level enforcement; IPs need L4 |
| Broad IP range (CIDR mask ≤ /16) | Manual review — could be Access app or Gateway L4 | Too broad for a single Access app; may need Gateway network policy instead |
| SCIM not configured | Start migration anyway, note identity rules for post-SCIM update | Don't block on SCIM — generate the structure, fill in identity later |
| HIP profiles present but posture not configured | Skip posture generation, document requirements | Posture rules need integration setup first (API keys, thresholds) |

## Implement

This section provides **three implementation paths** for applying the migration design to the customer's Cloudflare One account. Choose the path based on your tooling access:

| Path | When to Use | Prerequisites |
|------|------------|---------------|
| **Automated (Descaler)** | Full migration with many rules. Handles the entire pipeline — parsing, classification, object creation, and output generation — in one pass. | Access to a Descaler deployment |
| **MCP Tools** | Incremental migration, or when Descaler is unavailable. You drive the pipeline step-by-step using CF1 MCP tools. | MCP server connected with Gateway, Access, and Devices apps |
| **Dashboard / API** | No tooling available, or for manual review of individual rules. | CF dashboard access or API token with `Zero Trust: Edit` scope |

All three paths use the **same migration options** from the Assess section (`prefix`, `disabled`, `scim_configured`, `posture_configured`, etc.). When using MCP or Dashboard/API, you apply those options manually at each step — for example, prepending `prefix` to every object name, setting `enabled: false` when `disabled=true`, and including identity conditions only when `scim_configured=true`.

### Automated Migration (Descaler)

> **Descaler** is a Cloudflare Workers app for automated migration transforms. Deploy your own instance with `npm run build && npm run deploy`.

The Descaler automates the full 3-stage transform pipeline (Resolve → Classify → Map) and produces deployable output in multiple formats: JSON, Terraform (HELIX-compatible), Python deploy script, or direct API import.

1. Open the Descaler UI in your browser
2. Select **"Palo Alto Networks"** as the source vendor
3. Upload PAN-OS `set` commands or CSV/XLSX files (see Assess § 1 for formats)
4. Configure migration options in the UI:
   - **Prefix** — object name prefix (default: `descaler_`)
   - **Start disabled** — toggle on (recommended for first migration)
   - **SCIM configured** — toggle on only if SCIM is live between the customer's IdP and CF1
   - **Posture configured** — toggle on only if device posture integrations are set up
   - **Session duration** — Access app session length (default: 24h)
5. Click **Transform** → review the output configuration and all warnings
6. Choose export format:
   - **JSON** — raw `CloudflareConfig` for inspection or custom tooling
   - **Terraform** — HELIX-compatible `.tf` file using `var.account_id`, precedence 5000+
   - **Python script** — standalone `deploy.py` with dry-run support, reads credentials from env vars
   - **Direct API** — browser-based import via CF API proxy with progress tracking
7. **Review all warnings** before deploying — see Validate § 1 for warning taxonomy

### With MCP Tools (Code Mode)

Execute the migration in dependency order: Lists → Gateway rules → Access apps → Access policies → Device posture.

**Step 1: Create Gateway Lists (destination IPs and domains)**

For each security rule with specific destination IPs, a Gateway list is created:
```
→ search: "create gateway list" — find POST endpoint and request body schema
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/lists`,
    body: {
      name: "<prefix>panw_<rule_name>_dst",
      type: "IP",
      description: "Destination IPs from PANW rule \"<rule_name>\" [zones: <src> → <dst>]",
      items: [
        { value: "10.226.1.177/32" },
        { value: "10.226.1.0/24" }
      ]
    }
  })
```

For rules with FQDN destinations, create domain lists:
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/lists`,
    body: {
      name: "<prefix>panw_<rule_name>_domains",
      type: "DOMAIN",
      description: "Destination domains from PANW rule \"<rule_name>\"",
      items: [{ value: "app.corp.example.com" }]
    }
  })
```

**Step 2: Create Gateway Network L4 Rules**

For `network_allow` and `network_deny` classified rules:
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix>panw_l4_<rule_name>",
      description: "From PANW rule \"<rule_name>\" [zones: Trust → Untrust]",
      action: "allow",
      traffic: "net.dst.ip in $<list_id> and (net.dst.port == 443)",
      filters: ["l4"],
      precedence: 6000,
      enabled: false
    }
  })
```

When App-IDs have CF protocol mappings, add detected protocol:
```
"traffic_selector": "net.dst.ip in $<list_id> and net.detected_protocol in {\"ssh\"}"
```

When SCIM is configured, add identity selector:
```
"identity_selector": "identity.groups.name == \"VPN_OPS_IT\""
```

**Step 3: Create Gateway DNS Rules (for FQDN destinations)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix>panw_dns_<rule_name>",
      description: "DNS allow for PANW rule \"<rule_name>\"",
      action: "allow",
      traffic: "dns.fqdn in $<domain_list_id>",
      filters: ["dns"],
      precedence: 5000,
      enabled: false
    }
  })
```

**Step 4: Create Gateway HTTP Rules (for URL category rules)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix>panw_http_<rule_name>",
      description: "HTTP block for PANW rule \"<rule_name>\" [URL categories: adult, malware]",
      action: "block",
      traffic: "any(http.request.uri.content_category[*] in {<category_ids>})",
      filters: ["http"],
      precedence: 7000,
      enabled: false
    }
  })
```
> URL categories require manual mapping from PANW categories to CF security category IDs.

**Step 5: Create Access Applications (for identity_access rules)**

Access apps are deduped by destination+port. Multiple PANW rules targeting the same destination produce one app with multiple policies.
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/apps`,
    body: {
      name: "<prefix>panw_<rule_name>",
      type: "self_hosted",
      session_duration: "24h",
      destinations: [{ type: "private", uri: "10.226.1.0/24:443" }],
      auto_redirect_to_identity: true
    }
  })
```

For FQDN destinations, use hostname type:
```
destinations: [{ type: "private", uri: "app.corp.internal:8443" }]
```

**Step 6: Create Access Policies (for each identity rule)**
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/access/apps/${appId}/policies`,
    body: {
      name: "<prefix>policy_<rule_name>",
      decision: "allow",
      include: [{ group: { name: "VPN_OPS_IT" } }],
      require: [{ device_posture: { integration_uid: "<posture_rule_id>" } }]
    }
  })
```

If SCIM is not configured, use `everyone` include and note groups in the description:
```
body: {
  name: "<prefix>policy_<rule_name>",
  decision: "allow",
  include: [{ everyone: {} }],
  description: "[PANW identity: groups \"VPN_OPS_IT\" (domain: sso.corp.net)] — Configure SCIM and update this policy with group-based conditions."
}
```

**Step 7: Create Gateway L4 Catch-All for Access Apps**

When Access apps are generated, a catch-all Gateway L4 allow rule ensures Access-protected traffic can reach the apps through Gateway:
```
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/gateway/rules`,
    body: {
      name: "<prefix>panw_default_access_catchall",
      description: "Default Gateway L4 allow for traffic matching Access applications.",
      action: "allow",
      traffic: "net.dst.port > 0",
      filters: ["l4"],
      precedence: <last_network_precedence>,
      enabled: false
    }
  })
```

**Step 8: Create Device Posture Rules (from HIP profiles)**
```
→ search: "create device posture rule" — find the POST endpoint for posture rules
→ execute:
  cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/devices/posture`,
    body: {
      name: "<prefix>panw_posture_<hip_name>_disk-encryption",
      type: "disk_encryption",
      description: "Device posture: Disk encryption check [PANW HIP: <hip_name>]",
      match: [
        { platform: "windows" },
        { platform: "mac" },
        { platform: "linux" }
      ],
      schedule: "1h",
      expiration: "24h"
    }
  })
```

For EDR integrations (CrowdStrike, SentinelOne, Tanium), the posture rule is created but the s2s integration must be configured manually in the dashboard first.

### Without MCP Tools (Dashboard/API)

**Gateway Lists:**
1. Dashboard: Zero Trust → Gateway → Lists → Create a list
2. Select type (IP or Domain), add items from the resolved address/service objects
3. Name with prefix for identification

**Gateway Rules:**
1. Dashboard: Zero Trust → Gateway → Firewall Policies → Network (or DNS, HTTP)
2. Add a rule → configure name, traffic expression (wirefilter), action
3. Set precedence within the reserved ranges (DNS: 5000+, L4: 6000+, HTTP: 7000+)
4. Start with rules disabled

**API equivalent:**
```bash
# Create a Gateway list
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/lists" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "descaler_panw_ALLOW_WEB_dst",
    "type": "IP",
    "items": [{"value": "10.226.1.0/24"}]
  }'

# Create a Gateway Network rule
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "descaler_panw_l4_ALLOW_WEB",
    "action": "allow",
    "traffic": "net.dst.ip in $<list_id>",
    "filters": ["l4"],
    "precedence": 6000,
    "enabled": false
  }'
```

**Access Applications:**
1. Dashboard: Zero Trust → Access → Applications → Add an application → Self-hosted
2. Configure private destinations with IPs/CIDRs/hostnames from resolved addresses
3. Add policies with identity conditions (SCIM groups or email domain)

**Device Posture:**
1. Dashboard: Zero Trust → Settings → WARP Client → Device posture
2. Add checks matching HIP profile types (disk encryption, firewall, OS version)
3. For EDR integrations: Settings → WARP Client → Third-party integrations → configure API keys

## Validate

### 1. Review Warnings

Every migration produces warnings. Review them **before enabling any rules**:

- **`unmapped`** — Address/service/App-ID references that couldn't be resolved. Check if the source export is incomplete.
- **`unsupported`** — Disabled rules, schedule-based rules. Verify these should remain excluded.
- **`partial`** — URL categories that need manual CF mapping, HIP profiles that need integration configuration.
- **`manual_review`** — Broad IP ranges, identity rules without SCIM, rules that could be either Access apps or Gateway policies.

### 2. Verify Object Counts

Check that the migration produced the expected number of objects:

```
→ execute: GET `/accounts/${accountId}/gateway/lists`           — verify IP and domain lists (search for "<prefix>")
→ execute: GET `/accounts/${accountId}/gateway/rules`           — verify network/DNS/HTTP rules by filter type
→ execute: GET `/accounts/${accountId}/access/apps`             — verify Access applications
→ execute: GET `/accounts/${accountId}/devices/posture`         — verify posture rules
```

Cross-reference with the summary warning: `"Processed N PANW security rules: X identity-aware, Y disabled (skipped). Generated A Access apps, B Access policies, C Gateway network rules, D Gateway DNS rules, E Gateway HTTP rules."`

### 3. Test with Disabled Rules

All generated rules start disabled. Enable them one category at a time:

1. **Gateway DNS rules first** — lowest risk, verify domain resolution behavior
2. **Gateway Network L4 rules** — enable allow rules first, then block rules
3. **Gateway HTTP rules** — verify URL category mappings are correct
4. **Access applications + policies** — test with a pilot user group
5. **Device posture rules** — verify integration connectivity before enforcing

### 4. Identity Validation (if SCIM configured)

- Verify SCIM group names match exactly (case-sensitive) between IdP and CF1
- Test Access policies with users in each referenced LDAP group
- Check that Gateway wirefilter identity selectors resolve correctly

### 5. Edge Case Checks

- Rules with `destination=any` and `service=any` are skipped (would match all traffic) — verify this is correct
- Sweeping ranges (CIDR ≤ /16) marked for review — decide Access app vs Gateway L4
- `application-default` service resolution — verify App-ID ports are correct for your environment

## Operate

### Monitoring

- **Gateway activity logs**: Filter by rule name prefix to see migrated rule hits
- **Access audit logs**: Monitor authentication successes/failures for migrated Access apps
- **Device posture status**: Check device compliance rates for migrated posture rules
- **Warning review cadence**: Re-review `manual_review` warnings monthly as environment changes

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Migrated Gateway rule not matching traffic | Wirefilter expression incorrect, list reference wrong | Check `traffic` field; verify list ID is correct; test with `net.dst.ip` directly |
| Access app unreachable | Missing tunnel route for private destinations | Verify CIDR/hostname route exists on the tunnel serving this network |
| Identity policy not enforcing | SCIM group name mismatch (case-sensitive) | Compare IdP group names with SCIM sync; update policy `include` conditions |
| Posture rule failing all devices | Integration not configured (API keys missing) | Complete s2s integration setup in dashboard before enforcing posture rules |
| URL category rule too broad/narrow | PANW → CF category mapping mismatch | Review CF security category IDs; adjust `http.request.uri.content_category` selectors |
| Too many Gateway lists created | One list per rule with specific destinations | Consider consolidating lists for rules targeting the same network ranges |

### When to Revisit

- **Post-SCIM setup**: If migration ran with `scim_configured=false`, revisit all Access policies tagged `needs_identity` to add group-based conditions
- **Post-posture setup**: If migration ran with `posture_configured=false`, revisit all Access policies tagged `needs_posture` to add device posture requirements
- **Rule consolidation**: After initial validation, consider merging Gateway lists and rules targeting similar destinations
- **Precedence tuning**: Adjust precedence values if migrated rules conflict with existing policies

## Gotchas

1. **One PANW rule → multiple CF1 objects.** A single security rule can produce an Access application, an Access policy, a Gateway Network rule, a Gateway DNS rule, a Gateway HTTP rule, and one or more Gateway lists. Count your output objects, not your input rules.

2. **App-ID has no direct CF1 equivalent.** PANW App-IDs like `ms-teams`, `zoom`, `slack` are mapped to port-only matching because CF1's `net.detected_protocol` doesn't cover all applications. Unknown App-IDs generate `unmapped` warnings: `"App-ID '<name>' not found in Applipedia. Rule '<rule>' will use port-based matching only."` Review every unknown App-ID.

3. **LDAP groups require SCIM to be enforceable.** Without SCIM, identity-aware PANW rules produce Access policies with `include: [everyone]` and group names noted in the description only. These policies are **not** enforcing identity — they are placeholders. Tag: `needs_identity`.

4. **HIP profiles need integration setup before enforcement.** Device posture rules for EDR tools (CrowdStrike, SentinelOne, Tanium, Workspace ONE) require s2s integration configuration (API keys, tenant IDs) in the CF dashboard. The migration creates the rule structure but it won't work until the integration is live.

5. **Disabled rules are skipped entirely.** PANW rules with `disabled=yes` are classified as `unsupported` and produce only a warning. If you need to migrate disabled rules as disabled CF1 rules, you must modify the transform options.

6. **Schedule-based rules are unsupported.** PANW rules with a `schedule` field are skipped. CF1 Gateway rules support time-based schedules but the automated migration doesn't map them. These must be manually recreated.

7. **Broad IP ranges trigger manual review.** Any destination CIDR with mask ≤ /16 (e.g., `10.0.0.0/8`) is flagged with `user_choice_required`. These are too broad for a single Access app — consider splitting into multiple apps or using a Gateway L4 policy instead.

8. **URL category mapping is manual.** PANW URL categories (e.g., `adult`, `malware`, `hacking`) don't have 1:1 CF equivalents. The migration generates HTTP rules with placeholder traffic expressions. You must manually map to CF security category IDs.

9. **Circular address group references.** Address groups that reference each other (A → B → A) are detected and produce an `unsupported` warning. The circular members are excluded from resolution — check that the resulting list is complete.

10. **Unresolvable address/service references.** If a security rule references an address or service object that wasn't included in the export, an `unmapped` warning is generated. The reference is silently dropped from the resolved destination. Always export address objects/groups and service objects/groups alongside security rules.

11. **`destination=any` + `service=any` rules are skipped.** Rules matching all traffic are not migrated because the resulting Gateway rule would be a catch-all. A warning is generated: `"Rule has destination=any and service=any. Skipping — this would match all traffic."` Create these as explicit policy decisions in CF1.

12. **Access app destination limit is 500.** Each Access application supports up to 500 private destinations. Rules with very large resolved address groups may hit this limit — the excess is silently truncated. Split into multiple Access apps if needed.

13. **Domain join posture checks are Windows-only.** If a HIP profile has a domain join check but no Windows platform specified, the posture rule is skipped with a warning. Domain join is not applicable to Mac or Linux in Cloudflare.

14. **Generated rules start disabled by default.** This is intentional safety behavior (`disabled: true`). Enable rules in stages after validation. Do not bulk-enable all rules at once.

15. **Gateway L4 catch-all for Access traffic.** When Access apps are generated, a catch-all Gateway L4 allow rule (`net.dst.port > 0`) is appended at the end of the network precedence range. This ensures Access-protected traffic can transit through Gateway. Review whether this catch-all is too broad for your environment.

16. **Name truncation at 200 characters.** All generated object names are truncated to 200 characters. If PANW rule names are very long, the generated names may be cut off. Use shorter prefixes if name collisions occur.

## Related Skills

- **gateway-policy-design**: Design and refine Gateway policies (DNS/HTTP/Network) after migration. Use this to tune migrated rules, add URL category mappings, and optimize policy ordering.
- **access-app-setup**: Create and configure Access applications. Use this for detailed guidance on private destination setup, policy architecture, and session management for migrated apps.
- **device-posture-setup**: Configure device posture rules and integrations. Use this to complete the HIP profile → posture migration by setting up EDR integrations, thresholds, and compliance checks.
- **access-identity**: Set up SCIM provisioning and IdP configuration. **Critical dependency** — must be completed before identity-based PANW rules can be fully enforced in CF1.
- **tunnel-deployment**: Deploy Cloudflare Tunnels for private application connectivity. Access apps with private destinations require tunnel routes to reach the target network.
- **tunnel-routing**: Configure CIDR and hostname routes on tunnels. Complements Access app migration for private destinations.
- **migrate-zscaler-zia**: If the customer is also migrating from Zscaler ZIA alongside PANW, use this for internet security policy migration.
- **migrate-zscaler-zpa**: If the customer has both PANW and ZPA for private access, use this for ZPA-specific migration.
