---
name: "gateway-dlp"
description: "Design DLP profiles and integrate with Gateway HTTP policies for data loss prevention. Covers detection patterns, custom datasets, compliance templates (PCI, HIPAA, GDPR), and policy configuration."
domain: "gateway"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/policies/data-loss-prevention/"
---

# Gateway DLP Integration

## Persona

You are a senior Cloudflare One architect specializing in data loss prevention. You design DLP profiles that balance detection accuracy with false positive rates, and integrate them with Gateway HTTP policies for inline scanning.

## When to Use

- Customer needs to prevent sensitive data exfiltration (credit cards, SSNs, source code, etc.)
- Customer has compliance requirements (PCI-DSS, HIPAA, GDPR) that mandate DLP
- Customer wants to scan uploads/downloads for sensitive data patterns
- Customer asks about DLP profiles, datasets, or detection entries

## When NOT to Use

- Customer needs API-based SaaS data scanning (at rest) → use `casb-integration`
- Customer only needs content category filtering → use `gateway-policy-design`

## Assess

1. **What compliance frameworks apply?**
   - PCI-DSS (credit card data), HIPAA (health records), GDPR (personal data), SOX (financial records)?
   - Each framework maps to specific predefined DLP profiles.

2. **What data types need inline scanning?**
   - Credit card numbers, SSNs, health records, source code, API keys, internal document classifications?
   - Check existing profiles:
   → execute: GET `/accounts/${accountId}/dlp/profiles`

3. **Is TLS inspection enabled?**
   - DLP scanning requires TLS decryption of HTTPS traffic — hard prerequisite.
   - Check: GET `/accounts/${accountId}/gateway/configuration` → verify `tls_decrypt.enabled: true`
   - If not enabled, use `gateway-tls-inspection` skill first.

4. **What scanning direction is needed?**
   - Uploads only (exfiltration prevention), downloads only (ingestion control), or both?
   - This determines the traffic expression in Gateway HTTP policies.

5. **Does the customer need payload logging?**
   - Payload logging captures matched content for forensic review.
   - Requires a public key for encryption — must be configured before enabling.

## Design

DLP in Gateway works by attaching DLP profile references to Gateway HTTP policies. The profile defines **what** to detect; the policy defines **what action** to take.

### Architecture Pattern

```
DLP Profile (detection rules)  →  Gateway HTTP Policy (enforcement action)
         ↓                                    ↓
  Predefined entries                Block / Log / Allow
  Custom regex entries              + Payload logging
  Exact-match datasets              + Notification
```

### Key Design Decisions

| Decision | Recommendation |
|---|---|
| **Start with monitoring** | Use `action: "allow"` with payload logging enabled before switching to `block`. Tune for false positives first. |
| **Profile selection** | Use predefined profiles for standard data types (CC, SSN). Add custom profiles for org-specific patterns. |
| **Scope control** | Narrow traffic expressions to relevant hosts/paths. Scanning all HTTP traffic increases latency and false positives. |
| **Payload logging** | Enable for audit/compliance. Requires a public key — set up via Dashboard > Settings > DLP Payload Logging. |

### Prerequisites

- **TLS inspection** must be enabled (`gateway-tls-inspection`)
- **DLP profiles** must exist (`dlp-profile-design`)
- **Gateway HTTP policies** reference profiles via DLP-specific selectors

## Implement

### With MCP Tools (Code Mode)

1. Discover available DLP profiles:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/profiles` })
   — lists all DLP profiles with IDs for use in Gateway policies

2. Check TLS inspection is enabled (prerequisite):
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/configuration` })
   — DLP scanning requires TLS decryption

3. Get valid HTTP selectors for DLP policies:
   → search: "gateway rule selectors" — look for DLP-related selectors

4. Create a Gateway HTTP policy with DLP scanning:
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/gateway/rules`,
       body: {
         name: "Block DLP - Credit Cards",
         action: "block",
         traffic: 'any(http.request.uri.content_category[*] in {<dlp_profile_id>})',
         enabled: true,
         filters: ["http"],
         rule_settings: { payload_log: { enabled: true } }
       }
     })

5. Review existing DLP-related Gateway rules:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — identify rules that reference DLP profiles

### Without MCP Tools (Dashboard/API)

1. **Dashboard**: Zero Trust → Gateway → Firewall Policies → HTTP → Add a policy
2. Name the policy, set action to Block (or Allow for monitoring with payload logs)
3. In the traffic expression builder, select **DLP Profile** selector and choose the target profile
4. Under **Rule Settings**, enable Payload Logging if needed
5. Save and position the rule in the correct precedence order

```bash
# List DLP profiles (to find profile IDs)
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/dlp/profiles" \
  -H "Authorization: Bearer {api_token}" | jq '.result[] | {id, name, type}'

# Create a Gateway HTTP rule with DLP
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block DLP - Credit Cards",
    "action": "block",
    "traffic": "any(http.request.uri.content_category[*] in {<dlp_profile_id>})",
    "enabled": true,
    "filters": ["http"],
    "rule_settings": { "payload_log": { "enabled": true } }
  }'
```

## Validate

1. Verify DLP profiles are configured:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/profiles` })
   — confirm profiles exist and are active

2. Verify Gateway DLP rules are in place:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — check for HTTP rules with DLP-related traffic expressions

3. Inspect specific DLP Gateway rules:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules/${ruleId}` })
   — verify rule_settings include payload_log if needed

## Operate

1. **False positive blocking** — GET `/accounts/${accountId}/gateway/rules/${ruleId}` to inspect the DLP rule. Consider narrowing the traffic expression or adjusting DLP profile sensitivity.

2. **Update DLP policy scope** — PUT `/accounts/${accountId}/gateway/rules/${ruleId}` to modify traffic expressions (e.g., exclude specific hosts from DLP scanning).

3. **Review DLP profiles** — GET `/accounts/${accountId}/dlp/profiles` and GET `/accounts/${accountId}/dlp/profiles/${profileId}` to audit detection entries and confidence thresholds.

## Gotchas

1. **TLS inspection is a hard prerequisite.** DLP can only scan decrypted HTTPS content. If TLS inspection is off or a domain has a Do Not Inspect bypass, DLP won't detect sensitive data on that domain.

2. **Payload logging requires a public key.** Configure the encryption public key in Dashboard > Settings > DLP Payload Logging BEFORE enabling `payload_log` in rule_settings. Without it, payload logs won't capture matched content.

3. **Custom datasets have size limits.** Exact-match datasets vary by plan tier. Check current limits in docs before uploading large CSV files.

4. **DLP profiles are detection-only.** A profile with no Gateway HTTP policy referencing it does nothing. Always verify the profile is wired into an active HTTP rule.

5. **Monitor before blocking.** Start with `action: "allow"` + payload logging to identify false positives. Switch to `block` only after tuning.

## Related Skills

- **gateway-tls-inspection**: TLS inspection must be enabled for DLP to scan HTTPS traffic
- **gateway-policy-design**: DLP rules are implemented as Gateway HTTP policies with DLP profile selectors
- **dlp-profile-design**: Deep dive on DLP profile and dataset design (if you need more detail than this skill covers)
- **casb-integration**: API-based DLP scanning for data at rest in SaaS apps
