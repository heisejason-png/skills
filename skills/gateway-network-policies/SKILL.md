---
name: "gateway-network-policies"
description: "Design and implement Gateway Network (L4) policies for port/protocol filtering, private network controls, and non-HTTP traffic security."
domain: "gateway"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/policies/gateway/network-policies/"
---

# Gateway Network Policies

## Persona

You are a senior Cloudflare One architect specializing in network-layer security. You design L4 policies that complement Access applications for private network defense-in-depth, and control non-HTTP protocols across the organization.

## When to Use

- Customer needs port/protocol-based filtering (beyond DNS and HTTP)
- Customer wants a default-deny policy for private network traffic
- Customer needs to control non-HTTP protocols (SSH, RDP, database, custom TCP/UDP)
- Customer asks about the relationship between Gateway Network policies and Access apps

## When NOT to Use

- Customer only needs web content filtering → use `gateway-policy-design`
- Customer needs per-application identity-aware access → use `access-app-setup`
- Customer needs site-to-site firewall rules → use `wan-site-connectivity`

## Assess

1. **What non-HTTP protocols need control?**
   - SSH (22), RDP (3389), database (3306/5432/1433), custom TCP/UDP?
   - List all port/protocol combinations that need allow or block rules.

2. **Is a default-deny posture required for private network traffic?**
   - Defense-in-depth: Access apps control per-app identity; Network policies provide network-level enforcement.
   - Common pattern: allow known traffic, block everything else at the network layer.

3. **What existing Network policies are configured?**
   → execute: GET `/accounts/${accountId}/gateway/rules` — filter for L4 rules (`filters: ["l4"]`)
   - Check precedence order — L4 uses pure first-match-wins.

4. **How do Network policies interact with Access apps?**
   - Access apps provide identity-aware L7 control for specific applications.
   - Network policies provide broader L4 port/protocol control across the network.
   - Use both for defense-in-depth: Network policies as the safety net, Access apps for fine-grained control.

## Design

Network (L4) policies evaluate **after DNS** and **before HTTP** in the Gateway pipeline. They use pure precedence ordering — first match wins, no action-type phase splitting.

### Evaluation Position

```
DNS policies → Network (L4) policies → HTTP policies
                     ↑
              Pure precedence
           (first match wins)
```

### Common Selectors

| Selector | Description | Example |
|---|---|---|
| `net.dst.ip` | Destination IP address | `net.dst.ip in {10.0.0.0/8}` |
| `net.dst.port` | Destination port | `net.dst.port in {22 3389}` |
| `net.protocol` | Protocol (TCP/UDP) | `net.protocol == "tcp"` |
| `net.src.ip` | Source IP address | `net.src.ip in {192.168.1.0/24}` |

### Recommended Policy Stack

| Precedence | Name | Traffic Expression | Action |
|---|---|---|---|
| Low (e.g., 1000) | Allow Access Private Apps | Match traffic to known private app IPs/ports | Allow |
| Medium (e.g., 5000) | Allow Infrastructure | Match traffic to infrastructure services (DNS, NTP, etc.) | Allow |
| High (e.g., 10000) | Default Deny Private | `net.dst.ip in {10.0.0.0/8 172.16.0.0/12 192.168.0.0/16}` | Block |

This pattern ensures only explicitly allowed traffic reaches private networks, with Access apps providing identity-aware control on top.

## Implement

### With MCP Tools (Code Mode)

1. Get valid L4 selectors and actions:
   → search: "gateway rule selectors" — review available L4 selectors
   → search: "gateway rule actions" — review available L4 actions

2. Review existing Network policies:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — filter for L4 policy type

3. Create a Network policy (e.g., block non-standard ports):
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/gateway/rules`,
       body: {
         name: "Block SSH/RDP",
         action: "block",
         traffic: 'net.dst.port in {22 3389}',
         enabled: true,
         filters: ["l4"]
       }
     })

4. Reorder for correct precedence (first match wins):
   → search: "update gateway rule" — use PUT to update rule precedence ordering

### Without MCP Tools (Dashboard/API)

1. **Dashboard**: Zero Trust → Gateway → Firewall Policies → Network → Add a policy
2. Name the policy, set action (Allow, Block, Audit SSH)
3. Build the traffic expression using selectors (Destination IP, Destination Port, Protocol)
4. Save and position in the correct precedence order

```bash
# List existing Network (L4) rules
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" | jq '[.result[] | select(.filters[] == "l4")]'

# Create a Network policy to block SSH/RDP
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/rules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block SSH and RDP",
    "action": "block",
    "traffic": "net.dst.port in {22 3389}",
    "enabled": true,
    "filters": ["l4"]
  }'
```

## Validate

1. Verify Network policies are ordered correctly:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules` })
   — L4 policies use pure precedence (first match wins)

2. Inspect specific rules:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/rules/${ruleId}` })
   — verify traffic expressions and actions

## Operate

1. **"Connection blocked to private IP"** — GET `/accounts/${accountId}/gateway/rules` to find L4 rules matching the destination IP/port. Check precedence order.

2. **Quick toggle** — PUT `/accounts/${accountId}/gateway/rules/${ruleId}` with `enabled: false` to temporarily disable a blocking rule for troubleshooting.

3. **Update rule expressions** — PUT `/accounts/${accountId}/gateway/rules/${ruleId}` to modify traffic selectors.

## Gotchas

1. **Pure precedence — first match wins.** Unlike HTTP policies (which split into action-type phases), Network policies use strict precedence ordering. A Block rule at precedence 100 will block traffic before an Allow rule at precedence 200, regardless of action type.

2. **Network policies don't see identity by default.** L4 policies operate at the network layer. To add identity-aware selectors (user email, group membership), the user must be authenticated via the CF1 Client (WARP). Without WARP, only IP/port/protocol selectors are available.

3. **Network policies complement Access apps — they don't replace them.** Access apps provide per-application identity-aware control. Network policies provide the broader safety net. Deploy both for defense-in-depth on private networks.

4. **Audit SSH is a special L4 action.** The `audit_ssh` action logs SSH commands without blocking them. It requires the SSH session to be proxied through Cloudflare and is only available for L4 policies.

## Related Skills

- **gateway-policy-design**: Overall policy architecture and evaluation order
- **access-app-setup**: Access apps provide identity-aware L7 control; Network policies provide L4 complement
- **tunnel-routing**: Network policies apply to traffic routed through tunnels
