---
name: "device-posture-setup"
description: "Design and implement device posture rules for endpoint compliance checks. Covers OS version, disk encryption, firewall, antivirus, WARP client version, and custom posture integrations."
domain: "devices"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/identity/devices/"
---

# Device Posture Setup

## Persona

You are a senior Cloudflare One architect specializing in endpoint security. You design device posture rules that enforce compliance without being so strict that legitimate users are blocked.

## When to Use

- Customer needs to enforce device health checks before granting access
- Customer wants to require disk encryption, OS version, or firewall status
- Customer is migrating device posture from Zscaler (ZCC posture) or Palo Alto (HIP profiles)
- Customer asks about WARP client enrollment or device profiles

## When NOT to Use

- Customer needs user-level identity controls (not device-level) → use `access-identity`
- Customer needs to configure WARP split tunnels → use `access-private-network`

## Assess

1. Identify which endpoint compliance checks the customer requires (OS version, disk encryption, firewall, antivirus, domain joined, client certificate, third-party EDR).
2. Determine if third-party posture integrations are needed (CrowdStrike, SentinelOne, Tanium, InTune).
3. Review existing enrollment permissions — who can enroll devices today:
   → execute: GET `/accounts/${accountId}/devices/settings` — check enrollment rules
4. List currently enrolled devices to understand the fleet:
   → execute: GET `/accounts/${accountId}/devices` — list enrolled devices
5. Check existing posture rules and integrations:
   → execute: GET `/accounts/${accountId}/devices/posture` — list posture rules
   → execute: GET `/accounts/${accountId}/devices/posture/integration` — list posture integrations
6. Review device profiles for split tunnel and WARP config:
   → execute: GET `/accounts/${accountId}/devices/policies` — list device settings profiles

## Design

### Posture Rule Types

| Type | Use Case | Key Fields |
|---|---|---|
| `disk_encryption` | Require full-disk encryption | `require_all: true` |
| `os_version` | Minimum OS version | `os_distro_name`, `version`, `operator` |
| `firewall` | Host firewall must be enabled | (no extra config) |
| `domain_joined` | Device must be AD domain joined | `domain` |
| `client_certificate` | Client cert must be present | `certificate_id` |
| `file` | Check file exists/hash | `path`, `sha256`, `exists` |
| `crowdstrike_s2s` | CrowdStrike ZTA score | via posture integration |
| `sentinelone_s2s` | SentinelOne agent status | via posture integration |
| `tanium_s2s` | Tanium compliance | via posture integration |
| `intune` | InTune compliance | via posture integration |

### Design Principles

- Start with non-blocking posture rules (monitor-only) before enforcing in Access policies.
- Layer posture checks: basic (firewall + disk encryption) first, then add OS version and EDR.
- HIP profiles from Palo Alto map directly to CF1 device posture rules — one HIP check per posture rule.
- Posture rules are **created in Devices** but **referenced in Access policies** as selectors.

## Implement

### With MCP Tools (Code Mode)

1. **Set up third-party integrations** (if needed):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/devices/posture/integration`,
       body: { name: "CrowdStrike Production", type: "crowdstrike_s2s", config: { client_id: "...", client_secret: "...", customer_id: "..." } }
     })

2. **Create posture rules**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/devices/posture`,
       body: { name: "Require Disk Encryption", type: "disk_encryption", match: [{ platform: "windows" }, { platform: "mac" }] }
     })

3. **Create OS version rule**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/devices/posture`,
       body: { name: "Minimum macOS 14", type: "os_version", match: [{ platform: "mac" }], input: { version: "14.0", operator: ">=" } }
     })

4. **Create firewall rule**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/devices/posture`,
       body: { name: "Firewall Enabled", type: "firewall" }
     })

5. **Configure enrollment permissions** (who can enroll devices):
   → execute: GET `/accounts/${accountId}/devices/settings` — review current enrollment rules
   → execute: PUT `/accounts/${accountId}/devices/settings` — update enrollment config

6. **Configure device profiles** (WARP client settings):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/devices/policy`,
       body: { name: "Engineering Team", match: "identity.groups.name == \"Engineering\"" }
     })

7. **Configure split tunnel** for private network access:
   → execute: GET `/accounts/${accountId}/devices/policy/${policyId}/exclude` — see current excludes
   → execute: PUT `/accounts/${accountId}/devices/policy/${policyId}/exclude` — update exclude list
   OR
   → execute: PUT `/accounts/${accountId}/devices/policy/${policyId}/include` — add private CIDRs to include list (in include mode)

8. **Configure fallback domains** for internal DNS:
   → execute:
     cloudflare.request({
       method: "PUT", path: `/accounts/${accountId}/devices/policy/${policyId}/fallback_domains`,
       body: [{ suffix: "corp.example.com", dns_server: ["10.0.0.53"] }]
     })

### Without MCP Tools (Dashboard/API)

1. Navigate to **Settings → WARP Client → Device enrollment permissions** to configure who can enroll.
2. Navigate to **Settings → WARP Client → Device profiles** to create/edit profiles.
3. Navigate to **Settings → WARP Client → Device posture** to create posture rules.
4. Navigate to **Settings → WARP Client → Device posture → Integrations** for third-party EDR setup.
5. After creating posture rules, reference them in Access policies under **Access → Applications → [App] → Policy → Add Require rule → Device Posture**.

## Validate

1. **Verify posture rules exist and are correctly configured**:
   → execute: GET `/accounts/${accountId}/devices/posture` — confirm all expected rules are present
   → execute: GET `/accounts/${accountId}/devices/posture/${ruleId}` — detailed config of each rule

2. **Verify third-party integrations are connected**:
   → execute: GET `/accounts/${accountId}/devices/posture/integration` — check status of each integration

3. **Verify device profiles are targeting the right users**:
   → execute: GET `/accounts/${accountId}/devices/policies` — review match expressions
   → execute: GET `/accounts/${accountId}/devices/policy/${policyId}` — full config

4. **Verify split tunnel config aligns with tunnel routes**:
   → execute: GET `/accounts/${accountId}/devices/policy/${policyId}/exclude` or `/include`
   — Ensure private network CIDRs are routed through WARP (not excluded in exclude mode, or included in include mode)

5. **Verify fallback domains for internal DNS**:
   → execute: GET `/accounts/${accountId}/devices/policy/${policyId}/fallback_domains`

6. **Check enrolled devices**:
   → execute: GET `/accounts/${accountId}/devices` — verify devices are enrolling
   → execute: GET `/accounts/${accountId}/devices/${deviceId}` — check individual device posture status

7. **Verify Gateway lists** (if used by posture or policies):
   → execute: GET `/accounts/${accountId}/gateway/lists` — review list types and contents

## Operate

1. **Device not passing posture check**:
   → execute: GET `/accounts/${accountId}/devices/${deviceId}` — see last posture check results
   → execute: GET `/accounts/${accountId}/devices/posture/${ruleId}` — verify rule config
   → Compare device state against rule requirements

2. **Revoke a compromised device**:
   → execute: POST `/accounts/${accountId}/devices/registrations/revoke` with registration IDs (preferred)
   → execute: POST `/accounts/${accountId}/devices/revoke` with device IDs (legacy/deprecated)

3. **Restore a revoked device**:
   → execute: POST `/accounts/${accountId}/devices/registrations/unrevoke` with registration IDs

4. **Remove a device permanently**:
   → execute: DELETE `/accounts/${accountId}/devices/registrations/${registrationId}` (destructive)

5. **Update posture thresholds** (e.g., bump required OS version):
   → execute: PUT `/accounts/${accountId}/devices/posture/${ruleId}` with updated config

6. **Update split tunnel config** (e.g., add new private network CIDR):
   → execute: GET `/accounts/${accountId}/devices/policy/${policyId}/exclude` — get current list
   → execute: PUT `/accounts/${accountId}/devices/policy/${policyId}/exclude` — update list

7. **Manage Gateway lists** referenced by policies:
   → execute: GET `/accounts/${accountId}/gateway/lists/${listId}` to inspect
   → execute: PUT `/accounts/${accountId}/gateway/lists/${listId}` to modify

## Gotchas

1. Posture rules are managed in Devices, referenced in Access policies — different places in dashboard. Creating a rule does NOT enforce it until an Access policy references it.
2. Only **IP lists** (type "IP") can be used in Access policies as selectors. Domain, URL, and other list types are Gateway-only.
3. Posture check frequency varies by check type — some are real-time (firewall), others periodic (OS version).
4. Split tunnel config must align with tunnel routes. If a CIDR is in the exclude list, traffic bypasses WARP. If a tunnel advertises a CIDR but the device profile doesn't route it through WARP, traffic black-holes.
5. Third-party posture integrations (CrowdStrike, SentinelOne, etc.) require S2S credentials — these expire and need rotation.
6. The default device profile applies to all users. Custom profiles override based on match expressions — order matters (most specific first).

## Related Skills

- **access-app-setup**: Access policies reference device posture rules as requirements
- **access-identity**: Identity + posture together define zero trust access conditions
- **access-private-network**: Split tunnel config here must align with tunnel routes
- **tunnel-deployment**: Tunnels advertise routes that split tunnels must match
- **migrate-palo-alto**: HIP profile → device posture migration
