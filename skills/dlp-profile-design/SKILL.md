---
name: "dlp-profile-design"
description: "Design DLP profiles, detection entries, and custom datasets for sensitive data protection. Covers predefined profiles, custom regex patterns, exact-match datasets, and compliance-oriented configurations (PCI, HIPAA, GDPR)."
domain: "dlp"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/policies/data-loss-prevention/"
---

# DLP Profile Design

## Persona

You are a senior Cloudflare One architect specializing in data loss prevention. You design DLP profiles that maximize detection accuracy while minimizing false positives, tailored to the customer's compliance framework and data sensitivity requirements.

## When to Use

- Customer needs to design DLP detection profiles from scratch
- Customer has specific compliance requirements (PCI-DSS, HIPAA, GDPR)
- Customer wants to create custom detection patterns or exact-match datasets
- Customer asks about DLP profile structure, entries, or datasets

## When NOT to Use

- Customer needs to integrate DLP with Gateway policies → use `gateway-dlp`
- Customer needs API-based SaaS data scanning → use `casb-integration`

## Assess

1. **What data needs protection?**
   - Credit card numbers (PCI-DSS), health records (HIPAA), personal data (GDPR), source code, API keys, internal documents?
   - Each data type maps to a predefined or custom detection profile.

2. **What channels carry sensitive data?**
   - HTTP uploads/downloads (Gateway inline) → requires Gateway HTTP policies + TLS inspection
   - SaaS data at rest (API CASB) → requires CASB integration with DLP-enabled scan policy
   - Email (Email Security) → requires email security integration

3. **What detection accuracy is required?**
   - High-sensitivity environments (finance, healthcare) → tighter regex, lower thresholds, more detection entries
   - General corporate → predefined profiles are usually sufficient
   - Custom data formats (internal IDs, proprietary document markers) → custom regex or exact-match datasets

4. **What existing profiles are configured?**
   - Use GET `/accounts/${accountId}/dlp/profiles` to check current state before creating new profiles
   - Avoid duplicate profiles covering the same data types

## Design

DLP profiles define **what** to detect. They contain **detection entries** — individual patterns or datasets that match sensitive data.

### Profile Types

| Type | When to Use | Examples |
|------|------------|----------|
| **Predefined** | Standard compliance patterns maintained by Cloudflare | Credit Card Numbers, SSN, IBAN, API Keys, AWS Keys |
| **Custom** | Organization-specific data patterns | Internal employee IDs, proprietary document markers, custom regex |
| **Integration** | Third-party detection (Exact Data Match) | Uploaded CSV datasets for exact-match detection |

### Detection Entry Types

| Entry Type | How It Works | Best For |
|-----------|-------------|----------|
| **Predefined** | Cloudflare-maintained regex + validation | Standard data types (CC, SSN, IBAN) |
| **Custom regex** | User-defined regular expression | Organization-specific patterns |
| **Exact Data Match (EDM)** | Hash-match against uploaded dataset | Known sensitive values (customer IDs, account numbers) |

### Profile Design Principles

- **Start with predefined profiles** — they include validation logic (e.g., Luhn check for credit cards) that reduces false positives
- **Layer custom entries** on top of predefined profiles for org-specific patterns
- **Use exact-match datasets** for known sensitive values rather than regex approximations
- **Group related entries** in a single profile (e.g., "PCI Compliance" profile with CC + CVV + expiry entries)
- **Test with payload logs** before enforcing block actions — false positives in DLP are costly

## Implement

### With MCP Tools (Code Mode)

**Step 1: Review existing profiles and datasets**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/profiles` })
  — list all existing profiles (predefined and custom)
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/datasets` })
  — list existing exact-match datasets
```

**Step 2: Inspect profile details**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/profiles/${profileId}` })
  — see all detection entries within a profile, their types, and enabled status
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/entries` })
  — list all entries with their patterns and confidence levels
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/entries/${entryId}` })
  — inspect a specific entry's regex pattern, validation rules, and configuration
```

**Step 3: Inspect datasets**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/datasets/${datasetId}` })
  — see dataset metadata (upload status, column count, record count)
```

**Step 4: Check payload logs for detection activity**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/dlp/payload_log` })
  — review recent DLP detections (which profiles triggered, what content matched)
  — use this to validate profile accuracy and identify false positives
```

Note: Profile and entry creation is done via Dashboard or API. MCP tools are read-only for DLP configuration.

### Without MCP Tools (Dashboard)

1. **Dashboard**: Zero Trust → DLP → DLP Profiles
2. Click **Create profile** → choose Custom
3. Add detection entries:
   - Select from predefined entries (Credit Card, SSN, etc.)
   - Or create custom regex entries
4. Name the profile and save
5. **For exact-match datasets**: Zero Trust → DLP → Datasets → Upload CSV

### Without MCP Tools (API)

```bash
# List profiles
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/dlp/profiles" \
  -H "Authorization: Bearer {api_token}" | jq '.result'

# Get profile details
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/dlp/profiles/{profile_id}" \
  -H "Authorization: Bearer {api_token}" | jq '.result'
```

## Validate

1. **Verify profile structure:**
   - With MCP: GET `/accounts/${accountId}/dlp/profiles/${profileId}` — confirm entries are enabled and correctly configured
   - Check that each detection entry has the right confidence level and enabled status

2. **Test with payload logs:**
   - With MCP: GET `/accounts/${accountId}/dlp/payload_log` — check recent detections
   - Send test traffic containing known sensitive patterns through Gateway
   - Verify detections appear in payload logs with the correct profile and entry

3. **Check for false positives:**
   - Review payload logs for unexpected detections
   - Adjust regex patterns or add exclusions if false positive rate is too high

4. **Verify integration with enforcement:**
   - DLP profiles alone don't block anything — they must be referenced in Gateway HTTP policies (see `gateway-dlp`)
   - Or enabled on CASB integrations for data-at-rest scanning (see `casb-integration`)

## Operate

### Monitoring

- **Payload logs**: Regularly review GET `/accounts/${accountId}/dlp/payload_log` for detection patterns and false positives
- **Profile coverage**: When new data types need protection, add entries to existing profiles or create new ones
- **Dataset freshness**: Exact-match datasets must be re-uploaded when the source data changes

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| No DLP detections | Profiles not referenced in Gateway policies, or TLS inspection disabled | Check Gateway HTTP policy uses the profile. Verify TLS inspection is on (`gateway-tls-inspection`). |
| Too many false positives | Regex too broad, or predefined entry catching similar patterns | Review payload logs, tighten regex, increase confidence threshold, or add exclusion patterns |
| Dataset upload fails | CSV format incorrect, or file too large | Check column headers match expected format. Max dataset size varies by plan. |
| CASB DLP findings not appearing | DLP profiles exist but CASB integration not DLP-enabled | Re-create or update the CASB integration with a DLP-enabled scan policy |

## Gotchas

1. **DLP profiles are detection-only — they don't enforce anything alone.** Profiles must be referenced in Gateway HTTP policies (block/log actions) or enabled on CASB integrations. A profile with no policy reference is inert.

2. **TLS inspection is required for HTTPS DLP.** DLP can only scan decrypted content. If TLS inspection is off or excludes a domain, DLP won't detect sensitive data in HTTPS traffic to that domain.

3. **Predefined entries include validation logic.** Credit card detection uses Luhn check, not just regex. This reduces false positives significantly. Always prefer predefined entries over custom regex for standard data types.

4. **Exact-match datasets are hashed, not stored in plaintext.** The uploaded CSV is hashed and distributed. The original data is not stored by Cloudflare. Re-upload is required when source data changes.

5. **Payload logs have retention limits.** DLP payload logs are retained for a limited period. Export or review regularly for audit purposes.

## Related Skills

- **gateway-dlp**: Integrate DLP profiles with Gateway HTTP policies for inline enforcement
- **gateway-tls-inspection**: TLS inspection must be enabled for DLP to scan HTTPS traffic
- **casb-integration**: Enable DLP-based data-at-rest scanning on CASB integrations
