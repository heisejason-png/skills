---
name: "tunnel-routing"
description: "Configure tunnel routing: CIDR routes, hostname routes, virtual networks for overlapping IP spaces, and resolver policies for private DNS."
domain: "tunnel"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/tunnel/configure-tunnels/routing/"
---

# Tunnel Routing Configuration

## Persona

You are a senior Cloudflare One architect specializing in tunnel routing and private DNS. You configure routes that map private IP ranges and hostnames to the correct tunnels, and resolve DNS for private applications.

## When to Use

- Customer needs to configure CIDR or hostname routes on an existing tunnel
- Customer has overlapping IP spaces across environments
- Customer needs private DNS resolution (resolver policies or Local Domain Fallback)
- Customer asks about virtual networks or route priorities

## When NOT to Use

- Customer needs to deploy the tunnel itself → use `tunnel-deployment`
- Customer needs to create Access apps for the routed resources → use `access-app-setup`

## Assess

1. **What private CIDRs need routing?**
   - List all subnets that users need to reach (e.g., `10.0.0.0/8`, `172.16.0.0/16`).
   - Check existing routes:
   → execute: GET `/accounts/${accountId}/teamnet/routes`

2. **Are there overlapping IP spaces?**
   - Multiple environments (dev/staging/prod) sharing the same RFC 1918 ranges?
   - If yes, virtual networks are required to isolate each environment.
   → execute: GET `/accounts/${accountId}/teamnet/virtual_networks` — check existing virtual networks

3. **Does the customer need private hostname resolution?**
   - Private hostnames (e.g., `jira.corp.internal`) require explicit DNS configuration.
   - Two options: **Resolver policies** (recommended, managed in Gateway DNS) or **Local Domain Fallback** (device profile setting).

4. **What split tunnel mode is in use?**
   → execute: GET `/accounts/${accountId}/devices/policies` — identify the active profile
   - **Exclude mode**: target CIDRs must NOT be in the exclude list for traffic to route to Cloudflare.
   - **Include mode**: target CIDRs MUST be in the include list for traffic to route to Cloudflare.

5. **Which tunnels serve which networks?**
   → execute: GET `/accounts/${accountId}/cfd_tunnel` — list all tunnels
   → execute: GET `/accounts/${accountId}/teamnet/routes` — see current CIDR-to-tunnel mappings

## Design

### Route Types

| Route Type | Purpose | Key Behavior |
|---|---|---|
| **CIDR route** | Maps an IP range to a tunnel | Global scope — a CIDR maps to exactly ONE tunnel (unless scoped to a virtual network) |
| **Hostname route** | Maps a private hostname to a tunnel | Requires DNS resolution (resolver policy or LDF) to work |
| **Virtual network** | Isolates overlapping IP spaces | Must be assigned to both the tunnel route AND the CF1 Client device profile |

### Routing Architecture

```
CF1 Client (WARP) → Split Tunnel decision → Cloudflare Edge → Route lookup → Tunnel → Origin
                          ↑                                         ↑
                  Include/Exclude list                    CIDR or hostname route
                  on device profile                       scoped to virtual network
```

### Key Design Decisions

| Decision | Recommendation |
|---|---|
| **CIDR granularity** | Use the narrowest CIDRs that cover the required services. Avoid routing all of `10.0.0.0/8` if only `10.0.1.0/24` is needed. |
| **Overlapping IPs** | Create one virtual network per environment (prod, staging, dev). Assign routes and device profiles accordingly. |
| **Private DNS** | Use **resolver policies** (Gateway DNS) for centralized management. Use **Local Domain Fallback** only when Gateway DNS filtering is not desired. |
| **Route verification** | Use GET `/accounts/${accountId}/teamnet/routes/ip/<ip>` to confirm which tunnel serves a specific IP. |

## Implement

### With MCP Tools (Code Mode)

1. Understand the routing landscape:
   → execute: GET `/accounts/${accountId}/teamnet/routes` — see all CIDR routes across all tunnels
   → execute: GET `/accounts/${accountId}/teamnet/virtual_networks` — see all virtual networks
   → execute: GET `/accounts/${accountId}/cfd_tunnel` — find the tunnel UUIDs to associate routes with

2. Create a CIDR route (private network → tunnel):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/teamnet/routes`,
       body: { network: "10.0.0.0/8", tunnel_id: "<tunnel-uuid>", comment: "DC West subnet" }
     })
   - Optionally add `virtual_network_id` to scope to a virtual network

3. For overlapping CIDRs across environments, create virtual networks first:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/teamnet/virtual_networks`,
       body: { name: "prod", comment: "Production VPC" }
     })
   → Then create routes scoped to each virtual network

4. Create a hostname route (private hostname → tunnel):
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/zerotrust/routes/hostname`,
       body: { hostname: "app.internal", tunnel_id: "<tunnel-uuid>" }
     })
   - Optionally add `virtual_network_id`

5. Troubleshoot IP routing:
   → execute: GET `/accounts/${accountId}/teamnet/routes/ip/10.0.0.1` — shows which tunnel serves this IP

### Without MCP Tools (Dashboard/API)

1. Zero Trust Dashboard > Networks > Routes > Create route
2. Select tunnel, enter CIDR, optionally select virtual network
3. For hostname routes: Networks > Hostname Routes > Create

## Validate

1. Verify CIDR routes are correctly mapped:
   → execute: GET `/accounts/${accountId}/teamnet/routes` — check CIDRs, tunnel IDs, virtual network scoping
   → execute: GET `/accounts/${accountId}/teamnet/routes/ip/<sampleIp>` — confirm it resolves to the expected tunnel

2. Verify hostname routes:
   → execute: GET `/accounts/${accountId}/zerotrust/routes/hostname` — check hostnames are mapped to correct tunnels

3. Verify virtual networks:
   → execute: GET `/accounts/${accountId}/teamnet/virtual_networks` — ensure environments have separate virtual networks if CIDRs overlap

4. Verify the tunnel is healthy:
   → execute: GET `/accounts/${accountId}/cfd_tunnel/${tunnelId}` — confirm `status: "healthy"`

## Operate

1. **"Can't reach a private IP"** — GET `/accounts/${accountId}/teamnet/routes/ip/<ip>` to check if a route exists. If not, POST `/accounts/${accountId}/teamnet/routes` to create one. If it exists, GET `/accounts/${accountId}/cfd_tunnel/${tunnelId}` to check health.

2. **Update a route** — PATCH `/accounts/${accountId}/teamnet/routes/${routeId}` with updated JSON body.

3. **Delete a route** — DELETE `/accounts/${accountId}/teamnet/routes/${routeId}`. WARNING: traffic for the CIDR will no longer be routed.

4. **Manage virtual networks** — PATCH `/accounts/${accountId}/teamnet/virtual_networks/${vnId}` to rename, DELETE to remove (affects all routes in that network).

5. **Delete a hostname route** — DELETE `/accounts/${accountId}/zerotrust/routes/hostname/${hostnameRouteId}`.

## Gotchas

1. **CIDR routes are global by default.** A CIDR can only map to ONE tunnel unless you use virtual networks. Attempting to create a duplicate route for the same CIDR (without a different virtual network) will fail.

2. **Hostname routes require DNS resolution to work.** A hostname route alone does nothing — the client must be able to resolve the hostname to an IP. Configure a **resolver policy** in Gateway DNS or **Local Domain Fallback** on the device profile.

3. **Virtual networks must be assigned to BOTH tunnels AND device profiles.** Assigning a virtual network to a tunnel route but not the CF1 Client device profile (or vice versa) results in unreachable destinations. The device profile's default virtual network determines which route table the client uses.

4. **Split tunnel config must match routes.** If a CIDR has a tunnel route but is excluded from WARP traffic (Exclude mode) or not included (Include mode), the client won't send traffic to Cloudflare and the route is useless.

## Related Skills

- **tunnel-deployment**: Deploy the tunnel before configuring routes
- **access-private-network**: Full private access architecture using routes
