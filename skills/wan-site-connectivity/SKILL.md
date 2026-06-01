---
name: "wan-site-connectivity"
description: "Configure site-to-site WAN connectivity using Cloudflare WAN (Magic WAN). Covers IPsec/GRE tunnels, static routes, network firewall policies, appliance profiles, and analytics."
domain: "wan"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-wan/"
---

# WAN Site-to-Site Connectivity

## Persona

You are a senior Cloudflare One architect specializing in WAN connectivity. You design site-to-site architectures that replace MPLS and SD-WAN with Cloudflare's network, integrating with Gateway for security policy enforcement.

## When to Use

- Customer needs site-to-site connectivity between offices, data centers, or cloud regions
- Customer is replacing MPLS or legacy SD-WAN
- Customer needs network firewall policies between sites
- Customer asks about Magic WAN, IPsec tunnels, GRE tunnels, or CNI
- Customer has Cloudflare appliances (hardware connectors) to manage

## When NOT to Use

- Customer needs user-to-site remote access → use `access-private-network` + `tunnel-deployment`
- Customer only needs to connect one private network for remote access → use `tunnel-deployment`
- Customer wants WARP-client-based connectivity only → use `device-posture-setup`

## Assess

1. Identify customer's site topology — offices, data centers, cloud VPCs, and their geographic locations.
2. Determine on-ramp type per site:
   - **IPsec tunnel**: Most common, works with any router/firewall that speaks IKEv2.
   - **GRE tunnel**: Legacy option, less secure (no encryption by default).
   - **Appliance (mconn)**: Cloudflare managed hardware/virtual connector — auto-provisions tunnels.
   - **CNI**: Cloudflare Network Interconnect for high-bandwidth, private connectivity.
3. Inventory existing tunnels and routes:
   → execute: GET `/accounts/${accountId}/magic/gre_tunnels`
   → execute: GET `/accounts/${accountId}/magic/ipsec_tunnels`
   → execute: GET `/accounts/${accountId}/magic/routes`
4. Inventory existing sites and appliances:
   → execute: GET `/accounts/${accountId}/magic/sites`
   → execute: GET `/accounts/${accountId}/magic/connectors`
5. Check existing profiles (site configs):
   → search: "list magic sites" — find site sub-resource endpoints
6. Review existing network firewall rules:
   → execute: GET `/accounts/${accountId}/rulesets` — find ruleset with phase `magic_transit`
7. Check tunnel health baseline:
   → search: "magic transit tunnel health" — find health check analytics endpoints

## Design

### On-Ramp Selection

| On-Ramp | Use Case | Key Considerations |
|---|---|---|
| IPsec tunnel | Standard branch/DC connectivity | IKEv2, PSK or cert auth, health checks |
| GRE tunnel | Legacy or specific vendor requirements | Unencrypted, simpler setup |
| Appliance (mconn) | Managed hardware at branch | Auto-provisions tunnels/routes, LAN switching, DHCP |
| CNI | High-bandwidth DC/cloud | Private interconnect, no Internet transit |

### Architecture Principles

- CF WAN is ONLY for site-to-site connectivity between offices/branches. It is a connectivity method, NOT a security service.
- CF WAN and CF Tunnel (cloudflared) are **mutually exclusive** per destination network. Don't use both for the same subnet.
- Static routes define network reachability. Each route has prefix (CIDR), nexthop (tunnel interface IP), and priority (lower = higher).
- Routes with `managed_by: "bgp"` are BGP-learned and read-only. Routes with public ASN AS paths are likely Magic Transit (DDoS), not WAN.
- Network firewall (Wireshark-style expressions) filters inter-site traffic. This is NOT the same as Gateway policies (which filter WARP traffic).
- Profiles define the complete network config for appliances: WAN interfaces, LAN interfaces, ACLs, and traffic steering.

### Tunnel Redundancy

- Deploy **two IPsec tunnels per site** to different Cloudflare colos for redundancy.
- Use **priority** on static routes: primary route (priority 100) and backup (priority 200).
- Enable **bidirectional health checks** at mid or high rate for fast failover.

## Implement

### With MCP Tools (Code Mode)

#### A. IPsec Tunnel Deployment (most common)

1. **Create IPsec tunnels** for a branch site:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/ipsec_tunnels`,
       body: { ipsec_tunnels: [{ name: "branch-nyc-primary", customer_endpoint: "203.0.113.1", cloudflare_endpoint: "162.159.66.1", interface_address: "10.255.0.1/31", health_check: { rate: "mid", direction: "bidirectional" } }] }
     })

2. **Generate PSK** for the tunnel:
   → execute:
     cloudflare.request({ method: "POST", path: `/accounts/${accountId}/magic/ipsec_tunnels/${tunnelId}/psk_generate` })
   - Store the returned PSK securely — it cannot be retrieved again.

3. **Create static routes** for the branch subnets:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/routes`,
       body: { routes: [{ prefix: "10.1.0.0/16", nexthop: "10.255.0.1", priority: 100, description: "NYC branch primary" }] }
     })

4. **Create a WAN site** for topology visualization:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/sites`,
       body: { name: "NYC Branch", location: { lat: 40.7128, long: -74.0060 } }
     })

5. **Link the tunnel to the site** — configure via site WAN interface

#### B. GRE Tunnel Deployment

1. **Create GRE tunnels**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/gre_tunnels`,
       body: { gre_tunnels: [{ name: "dc-lon-primary", customer_gre_endpoint: "198.51.100.1", cloudflare_gre_endpoint: "162.159.66.2", interface_address: "10.255.1.1/31" }] }
     })

2. Create routes and site (same as IPsec steps 3-4).

#### C. Appliance (Connector) Deployment

1. **Register the connector**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/connectors`,
       body: { /* serial_number and config */ }
     })

2. **Create a site** for the connector:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/sites`,
       body: { name: "Branch NYC", connector_id: "<connector_id>" }
     })

3. **Add WAN interfaces** to the site:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/sites/${siteId}/wans`,
       body: { wan: { name: "WAN1", physport: 1, vlan_tag: 0, priority: 0, health_check_rate: "mid" } }
     })

4. **Add LAN interfaces**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/sites/${siteId}/lans`,
       body: { lan: { name: "LAN1", physport: 3, vlan_tag: 100, static_addressing: { address: "10.1.0.1/24" }, dhcp: { dhcp_server: { dhcp_pool_start: "10.1.0.100", dhcp_pool_end: "10.1.0.200", dns_server: "1.1.1.1" } } } }
     })

5. **Add LAN policies (ACLs)** between LAN segments:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/sites/${siteId}/acls`,
       body: { acl: { name: "Allow LAN1 to LAN2 HTTP", protocols: ["tcp"], lan_1: { lan_id: "<lan1_id>", ports: [80, 443] }, lan_2: { lan_id: "<lan2_id>" }, forward_locally: false } }
     })

6. **Configure traffic steering** (breakout/prioritization):
   → execute: GET `/accounts/${accountId}/magic/apps` — get available managed app IDs
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/magic/sites/${siteId}/app_configs`,
       body: { managed_app_id: "<app_id>", breakout: true }
     })

#### D. Network Firewall Rules

1. **List existing rules** to get the ruleset_id:
   → execute: GET `/accounts/${accountId}/rulesets` — find ruleset with phase `magic_transit`

2. **Create a firewall rule**:
   → execute:
     cloudflare.request({
       method: "POST", path: `/accounts/${accountId}/rulesets/${rulesetId}/rules`,
       body: { expression: "ip.src == 10.1.0.0/16 and tcp.dst == 443", action: "skip", description: "Allow HTTPS from NYC branch" }
     })

3. **Reorder rules** (rules are evaluated in order):
   → Update the ruleset with rules in desired order via PUT `/accounts/${accountId}/rulesets/${rulesetId}`

### Without MCP Tools (Dashboard/API)

1. Navigate to **Network → Tunnels** to create GRE or IPsec tunnels.
2. Navigate to **Network → Routes** to add static routes pointing to tunnel interfaces.
3. Navigate to **Network → Sites** to create site topology with location coordinates.
4. Navigate to **Network → Firewall** for Cloudflare Network Firewall rules (Wireshark-style expressions).
5. Navigate to **Network → Connectors** to register and manage appliances.
6. Navigate to **Network → Connector profiles** to configure WAN/LAN interfaces, ACLs, and traffic steering.

## Validate

1. **Verify tunnels are created and healthy**:
   → execute: GET `/accounts/${accountId}/magic/ipsec_tunnels` or `/magic/gre_tunnels` — confirm tunnels exist
   → Check health via Dashboard: Network → Tunnels → Health tab (≥95% = healthy, ≥50% = degraded, <50% = down)

2. **Verify routes are correct**:
   → execute: GET `/accounts/${accountId}/magic/routes` — confirm all site prefixes have routes with correct nexthops and priorities
   → execute: GET `/accounts/${accountId}/magic/routes/${routeId}` for detailed config

3. **Verify traffic is flowing**:
   → Check via Dashboard: Network → Analytics — look for ingress/egress traffic on tunnel interfaces

4. **Verify firewall rules**:
   → execute: GET `/accounts/${accountId}/rulesets` — find `magic_transit` phase ruleset, confirm rules exist with correct expressions and actions

5. **Verify connector health** (if applicable):
   → execute: GET `/accounts/${accountId}/magic/connectors/${connectorId}/telemetry/snapshots/latest` — check CPU, RAM, temperature, interface stats
   → execute: GET `/accounts/${accountId}/magic/connectors/${connectorId}/telemetry/events/latest` — recent lifecycle events

6. **Verify site config** (if applicable):
   → execute: GET `/accounts/${accountId}/magic/sites/${siteId}`
   → execute: GET `/accounts/${accountId}/magic/sites/${siteId}/wans`, `/lans`, `/acls` for detailed sub-configs

7. **Check for IDS detections**:
   → Check via Dashboard: Network → IDS Detections

## Operate

1. **Tunnel is down / degraded**:
   → execute: GET `/accounts/${accountId}/magic/ipsec_tunnels/${tunnelId}` — verify endpoint IPs and health check config
   → Check health via Dashboard: Network → Tunnels → Health tab

2. **Rotate IPsec PSK**:
   → execute:
     cloudflare.request({ method: "POST", path: `/accounts/${accountId}/magic/ipsec_tunnels/${tunnelId}/psk_generate` })
   - Coordinate with site admin to update CPE config simultaneously to minimize downtime.

3. **Add a new site/branch**:
   - Follow Implement steps A (IPsec) or C (Connector) above.

4. **Update static routes** (e.g., new subnet at existing site):
   → execute: POST `/accounts/${accountId}/magic/routes` with new prefix and nexthop
   OR
   → execute: PUT `/accounts/${accountId}/magic/routes/${routeId}` to modify existing route

5. **Connector troubleshooting**:
   → execute: GET `/accounts/${accountId}/magic/connectors/${connectorId}/telemetry/snapshots/latest` — check CPU load, temperature, RAM, interface errors
   → execute: GET `/accounts/${accountId}/magic/connectors/${connectorId}/telemetry/snapshots` — historical telemetry for trend analysis
   → execute: GET `/accounts/${accountId}/magic/connectors/${connectorId}/telemetry/events/latest` — lifecycle events (reboots, status changes)

6. **Firewall rule changes**:
   → execute: GET `/accounts/${accountId}/rulesets` — find `magic_transit` phase ruleset
   → execute: POST `/accounts/${accountId}/rulesets/${rulesetId}/rules` to create
   → execute: PATCH `/accounts/${accountId}/rulesets/${rulesetId}/rules/${ruleId}` to update
   → execute: DELETE `/accounts/${accountId}/rulesets/${rulesetId}/rules/${ruleId}` to delete
   → execute: PUT `/accounts/${accountId}/rulesets/${rulesetId}` to reorder (provide rules array in desired order)

7. **Investigate IDS detections**:
   → Check via Dashboard: Network → IDS Detections — filter by SignatureMessage or SourceIP

8. **Log-based troubleshooting**:
   → Check via Dashboard: Network → Analytics — L4 sessions through WAN tunnels, packet-level analytics

## Gotchas

1. CF WAN vs CF Tunnel — **mutually exclusive** per destination network. Don't route the same subnet through both WAN tunnels and cloudflared tunnels.
2. CF WAN is a **connectivity method**, NOT a security service. It sits alongside the Cloudflare platform, not inside it. Security is applied by Gateway policies and network firewall.
3. IPsec tunnel **MTU** is ~1476 bytes (1500 - IPsec overhead). May need MSS clamping on CPE to avoid fragmentation.
4. Network firewall rules use **Wireshark-style expressions** (e.g., `ip.src == 10.0.0.0/8`), NOT wirefilter expressions used by Gateway policies.
5. The API phase name is `magic_transit` even for WAN firewall rules — this is a naming artifact, not a product distinction.
6. BGP-managed routes (`managed_by: "bgp"`) are read-only. Don't try to update/delete them via API.
7. Routes with public ASN AS paths and /24+ public prefixes are **Magic Transit** (DDoS mitigation), not WAN routes. Different product, same tunnel infrastructure.
8. Appliance profiles use "site" as the API name. A profile contains WAN interfaces, LAN interfaces, ACLs, and app configs. Don't confuse with WAN sites (topology/location entities).
9. `POST /accounts/${accountId}/magic/ipsec_tunnels/${tunnelId}/psk_generate` returns the PSK **once**. It cannot be retrieved again — store it immediately.

## Related Skills

- **gateway-network-policies**: Network firewall policies apply to WAN tunnel traffic
- **tunnel-deployment**: Alternative to WAN for user-to-site connectivity (cloudflared)
- **device-posture-setup**: WARP client connectivity — different on-ramp than WAN
- **access-private-network**: Private network access for remote users over WARP/tunnels
- **migrate-palo-alto**: PANW SD-WAN → CF WAN migration
