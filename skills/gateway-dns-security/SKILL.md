---
name: "gateway-dns-security"
description: "Design and implement DNS filtering and security policies. Covers content categories, custom blocklists, safe search enforcement, and DNS-over-HTTPS configuration."
domain: "gateway"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/policies/gateway/dns-policies/"
---

# Gateway DNS Security

## Persona

You are a senior Cloudflare One architect specializing in DNS-layer security. You design DNS policies that provide broad security coverage as the first line of defense, complementing HTTP inspection for deeper content control.

## When to Use

- Customer wants DNS-based content filtering (categories, blocklists)
- Customer needs safe search enforcement across search engines
- Customer is setting up DNS security as a standalone deployment (no HTTP inspection)
- Customer asks about DNS policy selectors or resolver configuration

## When NOT to Use

- Customer needs HTTP content inspection or URL-based filtering → use `gateway-policy-design`
- Customer needs DLP scanning on web traffic → use `gateway-dlp`

## Assess

1. **What content categories need filtering?**
   - Security threats (malware, phishing, botnets, command & control)?
   - Content categories (adult, gambling, social media, streaming)?
   → execute: GET `/accounts/${accountId}/gateway/categories` — browse available categories and IDs

2. **Does the customer need safe search enforcement?**
   - Safe search forces search engines to return filtered results.
   - Requires DNS policy + may also require HTTP header injection for some engines.

3. **Is this DNS-only or will HTTP inspection follow?**
   - DNS-only is simpler: no TLS inspection, no certificate deployment, broad coverage.
   - If HTTP inspection is planned later, DNS policies provide the first layer while HTTP is rolled out.

4. **Are there custom domains to block or allow?**
   - Custom blocklists (competitor sites, known-bad domains) or allowlists (business-critical SaaS)?
   - These use Gateway Lists referenced in DNS policy expressions.

5. **What DNS locations exist?**
   → execute: GET `/accounts/${accountId}/gateway/locations` — check configured DNS resolver locations
   - Locations define which networks route DNS through Gateway.

6. **What existing DNS policies are configured?**
   → execute: GET `/accounts/${accountId}/gateway/rules` — filter for DNS rules (`filters: ["dns"]`)

## Design

DNS policies are the **first line of defense** in the Gateway pipeline — they evaluate before Network and HTTP policies.

### Evaluation Order

DNS policies have a unique two-phase evaluation that **ignores precedence**:
1. **Pre-resolution selectors** (`dns.domains`, `dns.fqdn`, content categories) — evaluated BEFORE the DNS query is resolved
2. **Post-resolution selectors** (`dns.resolved_ips`) — evaluated AFTER the DNS query is resolved

A pre-resolution Block rule at precedence 1000 will block BEFORE a post-resolution Allow rule at precedence 1, because pre-resolution always runs first.

### Selector Guidance

| Selector | Behavior | Recommendation |
|---|---|---|
| `dns.domains` | Splat field — matches domain AND all subdomains | **Recommended** for domain blocking/allowing |
| `dns.fqdn` | Exact match only — `example.com` does NOT match `sub.example.com` | Use only when exact-match precision is needed |
| `dns.content_category` | Matches Cloudflare content category IDs | Use for broad category filtering |
| `dns.security_category` | Matches Cloudflare security threat category IDs | Use for threat protection |
| `dns.resolved_ips` | Matches the resolved IP address (post-resolution) | Use for IP-based filtering |

### Common Policy Patterns

| Pattern | Traffic Expression | Action |
|---|---|---|
| Block all security threats | `any(dns.security_category[*] in {68 178 80 83})` | Block |
| Block adult content | `any(dns.content_category[*] in {<category_ids>})` | Block |
| Allow business-critical SaaS | `any(dns.domains[*] in $<allowlist_id>)` | Allow |
| Custom domain blocklist | `any(dns.domains[*] in $<blocklist_id>)` | Block |

## Implement

### With MCP Tools (Code Mode)

1. Discover DNS content and security categories:
   → search: "gateway categories" — find the categories endpoint
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/categories` })

2. Get valid DNS selectors:
   → search: "gateway rule selectors" — review available DNS selectors
   — use `dns.domains` (splat, recommended) not `dns.fqdn` (exact only)

3. Create a DNS security policy to block threat categories:
   → search: "create gateway rule" — find POST endpoint and request body schema
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/gateway/rules`,
       body: {
         name: "Block Threats",
         action: "block",
         traffic: 'any(dns.security_category[*] in {68 178 80 83})',
         enabled: true,
         filters: ["dns"]
       }
     })

4. Create a custom domain blocklist:
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/gateway/lists`,
       body: { name: "Custom Blocklist", type: "DOMAIN", items: [{ value: "bad.example.com" }] }
     })
   Then create a DNS rule referencing the list: `any(dns.domains[*] in $<list_id>)`

5. Add domains to an existing blocklist:
   → execute:
     cloudflare.request({
       method: "PATCH",
       path: `/accounts/${accountId}/gateway/lists/${listId}`,
       body: { append: [{ value: "new-bad.example.com" }] }
     })

6. Check DNS locations:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/locations` })

### Without MCP Tools (Dashboard/API)

1. **Dashboard**: Zero Trust → Gateway → Firewall Policies → DNS → Add a policy
2. Name the policy, set action (Block, Allow, Override, SafeSearch)
3. Build the traffic expression using the selector dropdown (Domain, Content Categories, Security Categories)
4. Save and verify precedence order

```bash
# List DNS categories
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/categories" \
  -H "Authorization: Bearer {api_token}" | jq '.result[] | {id, name, class}'

# Create a DNS block rule for security threats
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block Security Threats",
    "action": "block",
    "traffic": "any(dns.security_category[*] in {68 178 80 83})",
    "enabled": true,
    "filters": ["dns"]
  }'

# Create a custom domain blocklist
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/lists" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Custom Blocklist", "type": "DOMAIN", "items": [{"value": "bad.example.com"}]}'
```

## Validate

1. Verify DNS policies are active and ordered correctly:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — filter for DNS policy type, check precedence

2. Inspect a specific DNS policy:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules/${ruleId}` })
   — verify traffic expression includes correct category IDs or list references

3. Verify blocklist contents:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/lists/${listId}` })
   — check list items match expected domains

## Operate

1. **"Why is this domain blocked?"** — GET `/accounts/${accountId}/gateway/rules` to find DNS rules matching the domain. Check if it matches a category (GET `/accounts/${accountId}/gateway/categories`) or a list (GET `/accounts/${accountId}/gateway/lists/${listId}`).

2. **Quick unblock** — If a domain is in a custom list, PATCH the list to remove it. If blocked by category, either add an allow rule above it or toggle the rule with `enabled: false` via PUT.

3. **Add domains to blocklist** — PATCH `/accounts/${accountId}/gateway/lists/${listId}` with `append` items — changes propagate to all DNS rules referencing that list.

## Gotchas

1. **`dns.domains` (splat) vs `dns.fqdn` (exact match) — the #1 DNS policy mistake.** Use `dns.domains` to match a domain and all its subdomains. `dns.fqdn` matches ONLY the exact string, so blocking `example.com` won't block `sub.example.com`.

2. **Pre-resolution vs post-resolution evaluation ignores precedence.** All pre-resolution selectors (Domain, Host, Content Category) evaluate before ALL post-resolution selectors (Resolved IP), regardless of rule precedence numbers. A high-priority Allow on Resolved IP will NOT override a lower-priority Block on Domain.

3. **Safe search is more complex than it seems.** DNS-level safe search works for Google and Bing via CNAME override. Other engines may require HTTP header injection (`gateway-policy-design`), which requires TLS inspection.

4. **DNS policies don't see URL paths.** DNS operates at the domain level only. To filter by URL path or query parameters, use HTTP policies (`gateway-policy-design`).

## Related Skills

- **gateway-policy-design**: Overall Gateway policy architecture
- **gateway-tls-inspection**: Required for HTTP-layer content filtering beyond DNS
