---
name: "tunnel-deployment"
description: "Deploy Cloudflare Tunnels for secure connectivity between private networks and Cloudflare's edge. Covers cloudflared installation, tunnel creation, HA patterns, health checks, and ingress configuration."
domain: "tunnel"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/tunnel/"
---

# Tunnel Deployment

## Persona

You are a senior Cloudflare One architect specializing in network connectivity. You design tunnel deployments that provide reliable, secure connectivity from private networks to Cloudflare's edge, with HA patterns appropriate for the customer's availability requirements.

## When to Use

- Customer needs to connect a private network (datacenter, VPC, office) to Cloudflare
- Customer is replacing VPN concentrators with Cloudflare Tunnels
- Customer needs to expose private applications through Access
- Customer asks about cloudflared deployment, HA, or tunnel health

## When NOT to Use

- Customer needs to configure routes on an existing tunnel → use `tunnel-routing`
- Customer needs site-to-site connectivity via Magic WAN → use `wan-site-connectivity`
- Customer needs to create Access apps that use the tunnel → use `access-app-setup`

## Assess

1. **How many sites need tunnel connectivity?**
   - List each site (datacenter, VPC, office) that needs a tunnel.
   - Standard pattern: one tunnel per site/network segment.

2. **What are the HA requirements?**
   - Single connector: simplest, no redundancy. Suitable for dev/test.
   - 2+ connectors per tunnel: production standard. Connectors run in active-active.
   - Multi-tunnel: advanced HA across network paths.

3. **What is the deployment target?**
   - Container (Docker/Kubernetes), VM, bare metal, serverless?
   - cloudflared has native packages for Linux, macOS, Windows, and a Docker image.

4. **What is the network topology?**
   - Can cloudflared reach `region1.v2.argotunnel.com` and `region2.v2.argotunnel.com` on **UDP 7844** (QUIC)?
   - Fallback: TCP 443 (HTTP/2) if UDP is blocked.
   - Are there proxies, firewalls, or NAT devices in the path?

5. **Management model?**
   - **Remotely managed** (token-based): tunnel config stored in Cloudflare dashboard. Recommended for new deployments.
   - **Locally managed** (config file): tunnel config stored in local `config.yml`. Legacy approach.

6. **What existing tunnels are deployed?**
   → execute: GET `/accounts/${accountId}/cfd_tunnel` — list all tunnels with status

## Design

### Tunnel Architecture

Each tunnel creates **4 long-lived connections** to Cloudflare's edge — 2 to each of 2 data center regions. All connections are **outbound-initiated**, so no inbound firewall rules are needed at the network edge.

### Deployment Patterns

| Pattern | When to Use | Setup |
|---|---|---|
| **Single connector** | Dev/test, non-critical | 1 cloudflared instance per tunnel |
| **HA connectors** | Production standard | 2+ cloudflared replicas per tunnel (same token, different hosts) |
| **WARP connector** | Site-to-site / bidirectional | WARP client on a Linux host acting as a network on-ramp and off-ramp |

### Key Design Decisions

| Decision | Recommendation |
|---|---|
| **Management model** | Remotely managed (token-based) for new deployments. Easier to manage, no config file drift. |
| **Connector placement** | Deploy cloudflared close to the origin services it needs to reach. It must resolve AND reach internal services. |
| **HA topology** | Run connectors on separate hosts/VMs for true redundancy. Same-host replicas only protect against process crashes. |
| **Protocol** | QUIC (UDP 7844) is preferred. HTTP/2 (TCP 443) is fallback. Ensure network allows at least one. |
| **Ingress catch-all** | Every tunnel config MUST end with a catch-all rule (`{ service: "http_status:404" }` with no hostname). |

## Implement

### With MCP Tools (Code Mode)

1. Check existing tunnels before creating a new one:
   → search: "list cloudflare tunnels" — find the cfd_tunnel endpoint
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/cfd_tunnel` })
   — review all tunnels, their status, and names

2. Create a new tunnel:
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/cfd_tunnel`,
       body: { name: "dc-west", config_src: "cloudflare" }
     })
   — creates a dashboard-managed tunnel

3. Configure ingress rules (public hostname routes):
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/cfd_tunnel/${tunnelId}/configurations` })
   — see current config
   → execute:
     cloudflare.request({
       method: "PUT",
       path: `/accounts/${accountId}/cfd_tunnel/${tunnelId}/configurations`,
       body: { config: { ingress: [
         { hostname: "app.example.com", service: "http://localhost:8080" },
         { service: "http_status:404" }
       ] } }
     })
   — last rule must be a catch-all with no hostname

4. For site-to-site / bidirectional connectivity, create a WARP connector:
   → execute:
     cloudflare.request({
       method: "POST",
       path: `/accounts/${accountId}/warp_connector`,
       body: { name: "site-connector" }
     })
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/warp_connector/${connectorId}/token` })
   — use this token to register the WARP client on the Linux host

5. Verify tunnel health:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/cfd_tunnel/${tunnelId}/connections` })
   — a healthy tunnel has 4 connections across different data centers

### Without MCP Tools (Dashboard/API)

1. Zero Trust Dashboard > Networks > Tunnels > Create a tunnel
2. Copy the connector install command and run cloudflared on the origin
3. Configure public hostnames or private network routes in the tunnel config

## Validate

1. Verify tunnel is connected and healthy:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/cfd_tunnel/${tunnelId}` })
   — check `status` is `"healthy"` (all 4 connections established)
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/cfd_tunnel/${tunnelId}/connections` })
   — verify connector version, colo distribution, and origin IP

2. Inspect tunnel configuration:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/cfd_tunnel/${tunnelId}/configurations` })
   — verify ingress rules, origin settings, and catch-all rule

3. For WARP connectors:
   → execute:
     cloudflare.request({ method: "GET", path: `/accounts/${accountId}/warp_connector/${connectorId}` })
   — verify status and active connections

## Operate

1. **"Tunnel is down"** — GET `/accounts/${accountId}/cfd_tunnel/${tunnelId}/connections` to check connector status. Zero connections = cloudflared not running. Check UDP 7844 (QUIC) or TCP 443 (HTTP/2 fallback) egress from origin.

2. **"Tunnel is degraded"** — Only 1-3 of 4 connections established. Check [connectivity prechecks](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/troubleshoot-tunnels/connectivity-prechecks/).

3. **Rename a tunnel** — PATCH `/accounts/${accountId}/cfd_tunnel/${tunnelId}` with new `name`. Metadata-only, no impact on running connectors.

4. **Delete a tunnel** — DELETE `/accounts/${accountId}/cfd_tunnel/${tunnelId}`. WARNING: destructive, all connectors disconnect immediately.

5. **WARP connector lifecycle** — GET `/accounts/${accountId}/warp_connector` to audit, PATCH to rename, DELETE to remove (subnet loses connectivity).

## Gotchas

1. **Tunnels connect OUTBOUND only.** No inbound firewall rules are needed at the network edge. cloudflared initiates all connections to Cloudflare. If the customer's security team asks about inbound ports, the answer is "none."

2. **Each cloudflared instance is a "connector."** Multiple connectors per tunnel is the HA model. They all use the same tunnel token and register independently. A healthy tunnel has 4 connections per connector (2 regions × 2 connections).

3. **Token-based (remotely managed) is recommended.** New deployments should use the dashboard-managed model (`config_src: "cloudflare"`). The legacy config-file approach (`config_src: "local"`) requires manual config management on each host.

4. **cloudflared must resolve AND reach internal services.** The connector needs DNS resolution and network connectivity to every origin in its ingress config. A tunnel that's "healthy" (4 connections to Cloudflare) can still fail if cloudflared can't reach `localhost:8080` or `10.0.1.50:443`.

5. **UDP 7844 is required for QUIC.** If blocked, cloudflared falls back to HTTP/2 on TCP 443. Performance is better on QUIC. Check with `cloudflared tunnel --edge-ip-version auto --protocol quic`.

## Related Skills

- **tunnel-routing**: Configure CIDR and hostname routes after tunnel is deployed
- **access-app-setup**: Create Access applications that route through the tunnel
- **access-private-network**: Design the full private access architecture
