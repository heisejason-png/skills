---
name: "gateway-policy-design"
description: "Design and implement Gateway policies (DNS, HTTP, Network, Egress) for secure internet access. Covers policy ordering, wirefilter expressions, action selection, and integration with DLP, identity, and device posture."
domain: "gateway"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/policies/gateway/"
---

# Gateway Policy Design

## Persona

You are a senior Cloudflare One architect specializing in secure web gateway design. You understand the full Gateway policy evaluation pipeline and can design policies that balance security coverage with performance and user experience.

## When to Use

- Customer needs internet security (web filtering, DNS security, malware protection)
- Customer is designing a Gateway policy architecture from scratch
- Customer is migrating from a legacy SWG (Zscaler ZIA, Palo Alto, Netskope, etc.)
- Customer asks about policy ordering, precedence, or evaluation logic
- Customer needs to understand wirefilter expressions or selector options

## When NOT to Use

- Customer only needs private application access control → use `access-app-setup`
- Customer specifically needs TLS inspection configuration → use `gateway-tls-inspection`
- Customer specifically needs DLP policy setup → use `gateway-dlp`
- Customer needs site-to-site network firewall → use `wan-site-connectivity`

## Assess

1. **What is the user population?**
   - How many users? Remote-only, hybrid, or office-based?
   - Are there distinct user groups needing different policy sets (e.g., developers vs finance)?

2. **What existing SWG policies are in place?**
   - Migrating from Zscaler ZIA, Palo Alto, Netskope, or another SWG?
   - Export existing policy rules for mapping to Gateway equivalents.
   → Use `migrate-zscaler-zia` or `migrate-palo-alto` skills for structured migration.

3. **What compliance requirements apply?**
   - Content filtering mandates (CIPA, corporate AUP)?
   - DLP requirements (PCI-DSS, HIPAA, GDPR)? → use `gateway-dlp`
   - Logging/audit requirements for all web traffic?

4. **What identity sources are available?**
   - IdP configured for Gateway identity selectors? (Okta, Azure AD, Google Workspace)
   - SCIM provisioning enabled for group-based policies?
   → execute: GET `/accounts/${accountId}/access/identity_providers` — check IdP configuration

5. **What bypass requirements exist?**
   - Certificate-pinned apps needing Do Not Inspect rules? → use `gateway-tls-inspection`
   - Business-critical SaaS that must never be blocked?
   - Executive or VIP bypass groups?

## Design

Gateway policies are organized into four engines that evaluate in a fixed order. Within each engine, rules use precedence-based first-match-wins logic, with important exceptions for DNS and HTTP.

### Policy Evaluation Order

```
DNS → Egress → Network (L4) → HTTP
```

| Engine | Filters Value | Evaluation Behavior |
|---|---|---|
| **DNS** | `["dns"]` | Two-phase: pre-resolution selectors (`dns.domains`, categories) evaluate BEFORE post-resolution selectors (`dns.resolved_ips`), regardless of precedence. See `gateway-dns-security`. |
| **Egress** | `["egress"]` | Standard precedence (first match wins). Controls source IP for egress traffic. |
| **Network (L4)** | `["l4"]` | Standard precedence (first match wins). Port/protocol filtering. See `gateway-network-policies`. |
| **HTTP** | `["http"]` | Three action-type phases: (1) Do Not Inspect, (2) Isolate, (3) Allow/Block/Do Not Scan. Precedence only applies within each phase. See `gateway-tls-inspection`. |

### Decision Matrix

| Requirement | Policy Type | Why |
|---|---|---|
| Block malware/phishing domains | DNS | Broadest coverage, no TLS inspection needed, blocks at resolution |
| Block content categories (adult, gambling) | DNS | Category filtering at domain level, lightweight |
| URL path or query filtering | HTTP | DNS can't see URL paths — only HTTP policies inspect full URLs |
| DLP scanning (sensitive data) | HTTP | Requires TLS inspection + DLP profile integration |
| File type blocking | HTTP | Requires HTTP content inspection |
| Port/protocol control (SSH, RDP) | Network (L4) | Non-HTTP traffic, port-based filtering |
| Default-deny for private networks | Network (L4) | Broad network-level safety net |
| Source IP control for egress | Egress | Dedicated egress IP for specific destinations |

**General rule**: Start with DNS policies for broad coverage, add HTTP policies for deeper inspection, use Network policies for non-HTTP traffic control.

## Implement

### With MCP Tools (Code Mode)

1. Discover available selectors, actions, categories, and applications:
   → search: "gateway categories" — find content and security category IDs
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/categories` })
   → search: "gateway app types" — find recognized application IDs
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/app_types` })

2. Review existing policies before creating new ones:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — see all current rules with precedence order

3. Create a new policy:
   → search: "create gateway rule" — find POST endpoint and request body schema
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/gateway/rules`,
       body: {
         name: "Block Social Media",
         action: "block",
         traffic: 'any(dns.content_category[*] in {133 134})',
         enabled: true,
         filters: ["dns"]
       }
     })
   — `filters` determines policy type: `["dns"]`, `["http"]`, `["l4"]`, or `["egress"]`
   — Optional: `identity`, `device_posture`, `rule_settings`

4. Reorder policies for correct evaluation:
   → search: "update gateway rule" — use PUT to update rule precedence ordering

### Without MCP Tools (Dashboard/API)

1. **Dashboard**: Zero Trust → Gateway → Firewall Policies → select policy type tab (DNS, Network, HTTP)
2. Click **Add a policy** → name it, select action, build traffic expression
3. Use the expression builder for visual selector/operator/value selection
4. Or switch to **Expression editor** for raw wirefilter syntax
5. Save → verify precedence order in the policy list (drag to reorder)

```bash
# List all Gateway rules
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" | jq '.result[] | {id, name, action, filters, enabled, precedence}'

# List content and security categories
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/categories" \
  -H "Authorization: Bearer {api_token}" | jq '.result[] | {id, name, class}'

# Create a DNS block rule
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block Social Media",
    "action": "block",
    "traffic": "any(dns.content_category[*] in {133 134})",
    "enabled": true,
    "filters": ["dns"]
  }'

# Create a Gateway list
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/lists" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Allowed Domains", "type": "DOMAIN", "items": [{"value": "example.com"}]}'
```

## Validate

1. Verify all policies are in the correct order:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — check precedence values and enabled status

2. Inspect a specific policy's full configuration:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules/${ruleId}` })
   — verify traffic expression, action, and rule_settings

3. Verify lists referenced in policies contain expected entries:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/lists` })
   — find list IDs referenced in policy expressions

4. Confirm Gateway settings are properly configured:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/configuration` })
   — check TLS inspection, anti-virus, block page settings

## Operate

1. **"Why is this site blocked?"** — GET `/accounts/${accountId}/gateway/rules` to find rules matching the domain/URL, then GET `/accounts/${accountId}/gateway/lists` for any referenced lists. Explain which rule matched and why.

2. **Quick policy toggle** — PUT `/accounts/${accountId}/gateway/rules/${ruleId}` with `enabled: false` to disable a rule.

3. **Update an existing policy** — PUT `/accounts/${accountId}/gateway/rules/${ruleId}` to modify traffic expressions, actions, or rule_settings.

4. **Manage blocklists/allowlists** — PATCH `/accounts/${accountId}/gateway/lists/${listId}` to add or remove entries. Changes propagate to all rules referencing that list.

5. **Emergency policy removal** — DELETE `/accounts/${accountId}/gateway/rules/${ruleId}` to remove a problematic rule.

## Gotchas

1. **DNS pre/post-resolution evaluation ignores precedence.** Pre-resolution selectors (Domain, Content Category) always evaluate before post-resolution selectors (Resolved IP), regardless of rule precedence numbers. This is the most common source of "my Allow rule isn't working" issues.

2. **HTTP action-type phases override precedence.** Do Not Inspect rules (Phase 1) always run before Block/Allow rules (Phase 3), even if the Block rule has a lower precedence number. See `gateway-tls-inspection` for details.

3. **`dns.domains` (splat) vs `dns.fqdn` (exact match).** Use `dns.domains` to match a domain and all subdomains. `dns.fqdn` matches only the exact string. Blocking `dns.fqdn == "example.com"` will NOT block `sub.example.com`.

4. **Identity selectors require SCIM or IdP group sync.** Gateway policies can use identity selectors (user email, group) but only if the user is authenticated via CF1 Client (WARP) and the IdP is providing group claims via SCIM or SAML/OIDC attributes.

5. **`filters` determines policy type.** Setting `filters: ["dns"]` creates a DNS policy, `["http"]` creates HTTP, `["l4"]` creates Network, `["egress"]` creates Egress. Using the wrong filter is a silent misconfiguration — the rule won't match the traffic you expect.

6. **Policy ordering matters — first match wins (within phase).** If two rules could match the same traffic, the one with lower precedence number wins. Drag-and-drop in the Dashboard sets precedence.

## Related Skills

- **gateway-tls-inspection**: Configure TLS decryption (prerequisite for HTTP content inspection)
- **gateway-dns-security**: DNS-specific filtering and security policies
- **gateway-network-policies**: Layer 4 network policy design
- **gateway-dlp**: DLP profile integration with Gateway HTTP policies
- **access-app-setup**: Access apps complement Gateway for private network security
