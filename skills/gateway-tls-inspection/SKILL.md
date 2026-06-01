---
name: "gateway-tls-inspection"
description: "Configure TLS decryption (SSL inspection) for Gateway HTTP policy enforcement. Covers certificate deployment, Do Not Inspect rules, bypass strategies, and troubleshooting certificate errors."
domain: "gateway"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/policies/gateway/http-policies/tls-decryption/"
---

# Gateway TLS Inspection

## Persona

You are a senior Cloudflare One architect specializing in TLS inspection deployment. You know which applications break under inspection and how to design bypass rules that maintain security coverage while avoiding user-facing certificate errors.

## When to Use

- Customer is enabling HTTPS content inspection for the first time
- Customer has certificate errors after enabling Gateway
- Customer needs to design Do Not Inspect bypass rules
- Customer asks about TLS inspection coverage or limitations

## When NOT to Use

- Customer only needs DNS filtering (no HTTPS inspection required) → use `gateway-dns-security`
- Customer needs to design the overall Gateway policy architecture → use `gateway-policy-design`

## Assess

1. **How will the root CA be deployed?**
   - MDM (Intune, Jamf, Workspace ONE), Group Policy (GPO), manual installation?
   - The Cloudflare root CA must be trusted on ALL devices before enabling TLS inspection.
   - Download from: Dashboard > Settings > Resources > Certificates

2. **What certificate-pinned applications are in use?**
   - Common certificate-pinned apps: Slack desktop, Zoom, banking/financial apps, mobile apps, Windows Update.
   - These MUST have Do Not Inspect (DNI) bypass rules before enabling inspection.

3. **Are there compliance requirements?**
   - Some regulations require inspection of all HTTPS traffic for DLP/audit.
   - Others prohibit inspection of certain categories (banking, healthcare portals).
   - Design DNI rules to match compliance boundaries.

4. **Is FIPS mode required?**
   - FIPS mode forces TLS 1.2 only — breaks services that require TLS 1.3 or older TLS versions.
   - Only enable if the customer has explicit FIPS compliance requirements.

## Design

### HTTP Policy Action-Type Phases

HTTP policies evaluate in **three phases by action type**, not by precedence alone:

| Phase | Actions | Behavior |
|---|---|---|
| **Phase 1** | Do Not Inspect | Evaluated FIRST, regardless of precedence number. Traffic matching a DNI rule is never decrypted. |
| **Phase 2** | Isolate | Evaluated second. Matching traffic is sent to Browser Isolation. |
| **Phase 3** | Allow, Block, Do Not Scan | Evaluated last. Standard allow/block/log enforcement. |

**Precedence only applies WITHIN each phase.** A Block rule at precedence 1 will NOT override a Do Not Inspect rule at precedence 10000, because DNI always runs in Phase 1.

### Do Not Inspect Selector Rules

DNI rules run **before TLS decryption**, so only pre-decryption selectors are available:

| Selector | Use For | Example |
|---|---|---|
| `http.conn.server_name` | Domain/hostname bypass | `http.conn.server_name == "slack.com"` |
| `http.conn.server_ip` | IP-based bypass | `http.conn.server_ip in {1.2.3.4}` |
| Application selector | App-based bypass | Application matches "Slack" |

**CRITICAL: Do NOT use `http.request.host` in DNI rules.** This selector requires decrypted content, which isn't available in Phase 1. Use `http.conn.server_name` instead.

### Recommended DNI Bypass List

Start with these before enabling TLS inspection:
- Certificate-pinned apps (Slack, Zoom, Teams desktop clients)
- Banking and financial services
- Healthcare portals (HIPAA-sensitive)
- OS update services (Windows Update, Apple Software Update)
- Applications that use mutual TLS (mTLS)

## Implement

### With MCP Tools (Code Mode)

1. Check current TLS inspection settings:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/configuration` })
   — verify TLS decryption is enabled

2. Enable TLS inspection if not already on:
   → execute:
     cloudflare.request({
       method: "PUT",
       path: `/accounts/${accountId}/gateway/configuration`,
       body: { settings: { tls_decrypt: { enabled: true } } }
     })

3. Get valid selectors for Do Not Inspect rules:
   → search: "gateway rule selectors" — look for `http.conn.*` selectors (pre-inspection)

4. Review existing Do Not Inspect rules:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — filter for rules with `action: "off"` (Do Not Inspect)

5. Create Do Not Inspect bypass rules for certificate-pinned apps:
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/gateway/rules`,
       body: {
         name: "DNI - Certificate Pinned Apps",
         action: "off",
         traffic: 'http.conn.server_name == "app.example.com"',
         enabled: true,
         filters: ["http"]
       }
     })
   — use `http.conn.server_name` (NOT `http.request.host` — DNI runs before decryption)

### Without MCP Tools (Dashboard/API)

1. **Enable TLS inspection**: Zero Trust → Settings → Network → TLS Inspection → Enable
2. **Download root CA**: Zero Trust → Settings → Resources → Certificates → Download
3. **Deploy root CA** to all devices via MDM, GPO, or manual installation
4. **Create DNI rules**: Zero Trust → Gateway → Firewall Policies → HTTP → Add a policy → Action: Do Not Inspect
5. Use `SNI Domain` or `Application` selectors (pre-decryption) — NOT Host Header

```bash
# Check TLS inspection status
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/configuration" \
  -H "Authorization: Bearer {api_token}" | jq '.result.settings.tls_decrypt'

# Enable TLS inspection
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/configuration" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{"settings": {"tls_decrypt": {"enabled": true}}}'

# Create a Do Not Inspect rule
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "DNI - Certificate Pinned Apps",
    "action": "off",
    "traffic": "http.conn.server_name == \"slack.com\"",
    "enabled": true,
    "filters": ["http"]
  }'
```

## Validate

1. Verify TLS inspection is enabled:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/configuration` })
   — confirm TLS decryption setting

2. Verify Do Not Inspect rules are in place:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — DNI rules should appear (they run in Phase 1, before all other HTTP rules)

3. Inspect specific bypass rules:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules/${ruleId}` })
   — verify DNI rules use `http.conn.*` selectors, not `http.request.*`

## Operate

1. **Certificate errors reported** — GET `/accounts/${accountId}/gateway/rules` to check if the affected app has a DNI bypass rule. If not, POST a new rule with `action: "off"` using `http.conn.server_name`.

2. **Add emergency DNI bypass** — POST `/accounts/${accountId}/gateway/rules` with `action: "off"` and `http.conn.server_name == "affected-app.example.com"`. DNI rules take effect immediately in Phase 1.

3. **Modify existing bypass** — PUT `/accounts/${accountId}/gateway/rules/${ruleId}` to adjust traffic expressions on existing DNI rules.

## Gotchas

1. **Do Not Inspect rules evaluate in Phase 1 — always first.** No matter what precedence number you assign, DNI rules run before all Block, Allow, and Isolate rules. You cannot "override" a DNI rule with a higher-priority Block rule.

2. **Certificate-pinned apps MUST have DNI bypass rules.** Apps like Slack, Zoom, banking apps, and Windows Update use certificate pinning. Without DNI rules, users will see certificate errors and the apps will refuse to connect. Deploy DNI rules BEFORE enabling TLS inspection.

3. **Root CA must be deployed to ALL devices before enabling.** If TLS inspection is enabled before the Cloudflare root CA is trusted, every HTTPS connection will fail with certificate errors. Coordinate root CA deployment with your MDM/GPO rollout.

4. **FIPS mode forces TLS 1.2 only.** This breaks services that require TLS 1.3 or only support older TLS versions. Only enable FIPS mode if explicitly required by compliance.

5. **DNI selectors are limited to pre-decryption fields.** Only `http.conn.server_name`, `http.conn.server_ip`, and Application selectors work in DNI rules. HTTP request headers, URLs, and body content are NOT available because the traffic hasn't been decrypted yet.

## Related Skills

- **gateway-policy-design**: Overall Gateway policy architecture (parent skill)
- **device-posture-setup**: Root CA deployment often paired with device enrollment
