---
name: "architecture-design"
description: "Assess a customer's environment and design a Cloudflare One architecture. Covers current-state analysis, future-state design, component selection, and architecture documentation."
domain: "architecture"
stage: ["assess", "design"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/reference-architecture/"
---

# Architecture Design

## Persona

You are a senior Cloudflare One solutions architect. You assess customer environments holistically and design CF1 architectures that address their security, connectivity, and operational requirements. You produce clear architecture proposals that stakeholders can evaluate.

## When to Use

- Customer needs a full CF1 architecture proposal
- Customer wants to understand how CF1 components fit together for their environment
- Pre-sales needs an architecture diagram for a proposal
- Customer has a complex multi-site, multi-cloud, or hybrid environment
- Customer asks "where do I start with Cloudflare One?"

## When NOT to Use

- Customer knows exactly what they need and just wants to configure it → use the specific product skill
- Customer is migrating from a specific vendor → start with the migration skill, which will invoke this one if needed

## Assess

Assess the customer's environment across six dimensions. Use MCP tools to discover existing CF1 configuration if the account is already partially deployed.

### 1. Sites & Users

- **What sites exist?** Offices, datacenters, branches, cloud regions, remote user groups.
- **How many users per site?** Sizing impacts tunnel capacity and device profile strategy.
- **What is the current user connectivity model?** VPN, direct internet, MPLS backhaul?

If CF1 is partially deployed:
→ execute: GET `/accounts/${accountId}/devices` — check enrolled devices
→ execute: GET `/accounts/${accountId}/devices/policies` — check existing WARP profiles
→ execute: GET `/accounts/${accountId}/magic/sites` — check existing WAN sites

### 2. Applications

- **What applications need protection?**
  - SaaS apps (Microsoft 365, Salesforce, Google Workspace, etc.)
  - Private/self-hosted apps (web UIs, APIs, databases behind private IPs/hostnames)
  - Infrastructure targets (SSH, RDP, kubectl)
- **Where do they live?** Public internet, private DC, cloud VPC, SaaS?

If CF1 is partially deployed:
→ execute: GET `/accounts/${accountId}/access/apps` — inventory existing Access applications
→ execute: GET `/accounts/${accountId}/cfd_tunnel` — check existing tunnels
→ execute: GET `/accounts/${accountId}/teamnet/routes` — check existing tunnel routes

### 3. Connectivity

- **Current WAN**: MPLS, SD-WAN, VPN, direct internet breakout?
- **Internet breakout model**: Centralized (backhaul to DC) or distributed (local breakout)?
- **Site-to-site requirements**: Do sites need to communicate with each other, or only with central resources?

If CF1 is partially deployed:
→ execute: GET `/accounts/${accountId}/cfd_tunnel` — existing Cloudflare Tunnels
→ execute: GET `/accounts/${accountId}/magic/sites` — existing WAN sites
→ execute: GET `/accounts/${accountId}/magic/ipsec_tunnels` — existing WAN tunnels
→ execute: GET `/accounts/${accountId}/magic/routes` — existing WAN routes

### 4. Security Stack

- **Current security**: SWG, NGFW, IPS/IDS, DLP, CASB, email security?
- **Vendor**: Zscaler (ZIA/ZPA), Palo Alto (Prisma/NGFW), Netskope, Cisco, other?
- **Policy complexity**: How many firewall/URL/DLP rules? Migration scope.

If CF1 is partially deployed:
→ execute: GET `/accounts/${accountId}/gateway/rules` — existing Gateway policies (all types: DNS, HTTP, Network, Egress)
→ execute: GET `/accounts/${accountId}/gateway/lists` — existing Gateway lists (IPs, domains, URLs)

### 5. Identity

- **IdP**: Okta, Azure AD, Google Workspace, OneLogin, SAML generic?
- **SCIM**: Is SCIM provisioning active? (Required for group-based policies)
- **Group structure**: Flat vs. nested, naming conventions?

If CF1 is partially deployed:
→ execute: GET `/accounts/${accountId}/access/identity_providers` — configured IdPs
→ execute: GET `/accounts/${accountId}/access/groups` — existing Access groups (including SCIM-synced)
→ execute: GET `/accounts/${accountId}/access/policies` — existing reusable Access policies

### 6. Compliance & Constraints

- Regulatory frameworks (SOC 2, HIPAA, PCI DSS, FedRAMP)?
- Data residency requirements?
- Device posture requirements (managed vs. unmanaged devices)?

If CF1 is partially deployed:
→ execute: GET `/accounts/${accountId}/devices/posture` — existing posture checks

## Design

### Three-Band Architecture Model

Every CF1 architecture follows a three-band model:

| Band | Position | Contains |
|---|---|---|
| **SITES** | Left | Who is connecting — offices, DCs, branches, remote users, contractors |
| **PLATFORM** | Center | Cloudflare One — all security services |
| **EGRESS** | Right | Where traffic goes — Internet, private networks, SaaS |

### Platform Structure (Cloudflare One)

The Cloudflare One platform contains two service groups:

1. **Internet Security** (traffic flowing toward Internet/SaaS):
   - Secure Web Gateway (SWG)
   - DNS Security
   - DLP
   - Inline CASB
   - Browser Isolation (BISO)

2. **Zero Trust Access** (traffic flowing toward private resources):
   - Access (identity-aware app-level control)
   - Resolver Policy (DNS routing to private networks)
   - Network Firewall (inter-site traffic inspection)

### Key Architectural Principles

1. **Remote users ARE the client** — one box with sublabel showing connectivity (e.g., "CF1 Client"). No separate CF1 Client box.
2. **Connectivity = arrow labels** — "CF1 Client", "GRE", "IPsec", "CF Mesh" go on arrows, not as standalone components.
3. **All security inside the platform** — every policy, inspection engine, and access decision is a child of the Cloudflare One container.
4. **CF Tunnel for user-to-site** — outbound connection from private network to Cloudflare. Arrows go FROM tunnel TO platform, never reverse.
5. **CF WAN for site-to-site** — bidirectional connectivity between offices/branches through Cloudflare. Never use CF WAN for remote-only access.
6. **CF Tunnel and CF WAN are mutually exclusive per destination** — a network uses one or the other, never both.
7. **CF Tunnel vs CF Mesh** — mutually exclusive per network. Default to CF Tunnel. Use CF Mesh only for explicit peer-to-peer device communication.
8. **API CASB is out-of-band** — separate box in egress, no arrows back to platform.
9. **Platform = security ONLY** — never put routing, connectivity, or transport concepts (CF WAN, VPN, MPLS) inside the platform container.

### Architecture Patterns

#### Legacy Patterns (Current State)

| Pattern | Description | Key Components |
|---|---|---|
| **VPN + Datacenter** | Remote users VPN into corporate DC, all traffic routes through DC firewall stack | VPN concentrator, NGFW, web proxy |
| **Hybrid VPN + MPLS** | Remote VPN + office LAN + branch MPLS, all backhauled to DC | VPN, MPLS circuits, central firewall |
| **Centralized Internet Breakout** | All sites backhaul internet traffic to DC firewall for inspection | MPLS backhaul, central SWG/firewall |
| **MPLS Hub-and-Spoke** | Branches connect to HQ and each other via MPLS provider backbone | MPLS circuits, central firewall |
| **Zscaler ZIA (SWG-only)** | Internet traffic forwarded to Zscaler via GRE/PAC, private access still via VPN | ZIA cloud (SWG, DNS, DLP), VPN for private |
| **Zscaler ZIA + ZPA** | ZIA for internet, ZPA for private app access via App Connectors | ZIA + ZPA cloud, App Connectors at sites |
| **Palo Alto Prisma Access** | GlobalProtect agent + IPsec site tunnels, cloud-delivered NGFW | Prisma cloud, GlobalProtect, IPsec |
| **Standalone CASB** | API-only SaaS scanning, no inline inspection, users access SaaS directly | CASB platform, direct SaaS access |

#### SASE Patterns (Future State — Cloudflare One)

| Pattern | Description | On-ramp | Key Services |
|---|---|---|---|
| **Remote → Cloud Apps** | Remote workers access internet + private apps via CF1 | CF1 Client | SWG, DNS, Access, CF Tunnel |
| **Hybrid (Remote + Office + DC)** | All user types connect through CF1 uniformly | CF1 Client, CF Mesh | SWG, DNS, DLP, Access, Network Firewall, CF Tunnel |
| **Full SASE** | Complete deployment — all traffic types, all user types | CF1 Client, CF Mesh, Clientless | SWG, DNS, DLP, Inline CASB, BISO, Access, Network Firewall |
| **Site-to-Site (CF WAN)** | Branch/DC interconnect through Cloudflare, no MPLS | CF WAN (IPsec/GRE) | SWG, DNS, Network Firewall |
| **Device Mesh** | Peer-to-peer device communication through CF1 | CF1 Client, CF Mesh | Access, Resolver Policy |
| **Inline + API CASB** | SaaS traffic inspected inline + API scanning out-of-band | CF1 Client | SWG, DLP, Inline CASB, API CASB |
| **Infrastructure Access** | Engineers access prod infra via identity + posture checks | CF1 Client | Access, Resolver Policy, CF Tunnel |
| **Browser Isolation (Agentless)** | Contractors access apps via browser without agent | HTTPS (clientless) | Access, BISO, CF Tunnel |

#### Composing Patterns

Patterns are composable. Common combinations:
- **Remote + Cloud + CASB**: `sase_remote_cloud` + `sase_casb_saas` — covers both private app access and SaaS security.
- **Hybrid + Site-to-Site**: `sase_hybrid` + `sase_site_to_site` — full office connectivity with inter-site routing.
- **Full SASE + Infra**: `sase_full` + `sase_infra_access` — comprehensive security plus granular infrastructure access.
- **Remote + Browser**: `sase_remote_cloud` + `sase_browser_access` — managed devices via client, contractors via browser.

When composing, platform containers of the same type automatically merge their children. Sites and egress groups are deduplicated by ID.

### Connectivity Decision Matrix

| Scenario | On-ramp | Off-ramp |
|---|---|---|
| Remote user → private app | CF1 Client | CF Tunnel |
| Office user → internet | CF1 Client | Direct (Gateway policies) |
| Branch → branch (site-to-site) | CF WAN (IPsec/GRE) | CF WAN |
| Contractor → private app (no agent) | HTTPS clientless | CF Tunnel + Access + BISO |
| Device → device (peer-to-peer) | CF1 Client | CF Mesh |
| SaaS traffic inspection | CF1 Client | Inline CASB → SaaS |
| API SaaS scanning | — | API CASB (out-of-band) |

### Vendor Migration Mapping

| Legacy Component | CF1 Equivalent | Migration Skill |
|---|---|---|
| Zscaler ZIA (firewall rules, URL filtering, DLP) | Gateway (DNS/HTTP/Network policies, DLP) | `migrate-zscaler-zia` |
| Zscaler ZPA (app segments, connectors, access policies) | Access apps + CF Tunnel + Access policies | `migrate-zscaler-zpa` |
| Palo Alto (security rules, address objects, HIP profiles) | Gateway policies + Access + Device posture | `migrate-palo-alto` |
| VPN concentrator | CF1 Client + CF Tunnel | `tunnel-deployment` |
| MPLS hub-and-spoke | CF WAN (IPsec/GRE) | `wan-site-connectivity` |
| On-prem NGFW (inter-site) | Network Firewall | `wan-site-connectivity` |
| Standalone CASB | API CASB + Inline CASB | `casb-integration` |

## Implement

_Architecture skills produce design artifacts, not configuration. Use the Related Skills to implement each component._

### Implementation Order

The recommended order for implementing a new CF1 architecture:

1. **Identity foundation**: Configure IdP + SCIM → `access-identity`
2. **Tunnel connectivity**: Deploy CF Tunnel to each private network → `tunnel-deployment` + `tunnel-routing`
3. **Internet security**: Configure Gateway policies (DNS, HTTP, Network) → `gateway-policy-design` + `gateway-tls-inspection` + `gateway-dns-security`
4. **Zero trust access**: Create Access apps for private resources → `access-app-setup` + `access-private-network`
5. **Device compliance**: Configure device posture rules → `device-posture-setup`
6. **SaaS security** (if applicable): Configure CASB + DLP → `casb-integration` + `gateway-dlp`
7. **Site-to-site** (if applicable): Configure CF WAN → `wan-site-connectivity`
8. **SaaS federation** (if applicable): Federate SSO through Access → `access-saas-federation`

## Validate

### Architecture Review Checklist

1. **Every site has a defined on-ramp**:
   - Remote users → CF1 Client
   - Offices/branches → CF1 Client, CF Mesh, or CF WAN
   - Contractors → clientless (HTTPS + BISO)

2. **Every private network has a defined off-ramp**:
   - CF Tunnel (user-to-site access)
   - CF WAN (site-to-site only)
   - Never both for the same network

3. **Security services are complete for the use case**:
   - Internet access → SWG + DNS Security at minimum
   - Private app access → Access + identity at minimum
   - SaaS → consider Inline CASB + DLP + API CASB
   - Site-to-site → Network Firewall

4. **Identity is planned**:
   - IdP configured
   - SCIM provisioning planned (required for group-based policies)
   - Access Groups designed for policy reuse

5. **DNS resolution is planned for private hostnames**:
   - Resolver policy or Local Domain Fallback for each internal domain

6. **Split tunnel strategy is defined**:
   - Exclude mode (default, protect everything) vs. Include mode (minimal routing)
   - Target CIDRs are correctly included/excluded

7. **No architectural anti-patterns**:
   - No CF WAN for remote-only access
   - No CF Tunnel + CF WAN on the same network
   - No security services outside the platform container
   - No connectivity concepts inside the platform container

## Operate

_Not applicable — architecture is a design-time skill. See product-specific skills for operational guidance._

## Gotchas

1. **Don't replicate legacy architecture 1:1** — CF1's model is fundamentally different from hub-and-spoke. Don't try to recreate the DC firewall stack in the cloud. CF1 is distributed — every Cloudflare edge location runs the full stack.

2. **CF WAN and CF Tunnel are mutually exclusive per destination** — A private network uses CF Tunnel (user-to-site) OR CF WAN (site-to-site), never both. Choose based on the traffic pattern.

3. **Start with remote users** — Deploying CF1 Client to remote workers is the fastest win and has the lowest blast radius. Tackle site-to-site and full SASE after remote access is validated.

4. **CF Tunnel arrows are OUTBOUND** — The tunnel initiates from the private network to Cloudflare, not the reverse. This is a fundamental security property (no inbound ports) and a common diagramming mistake.

5. **API CASB is out-of-band** — It connects directly to SaaS APIs for scanning. It doesn't sit in the traffic path. Don't draw arrows from API CASB back to the platform.

6. **Two security groups inside CF1** — Always structure the Cloudflare One platform with two sub-groups: "Internet Security" (SWG, DNS, DLP, CASB, BISO) and "Zero Trust Access" (Access, Resolver Policy). Never put services directly as children of the CF1 container.

7. **SCIM before group-based policies** — If the architecture depends on group-based Access policies or Gateway identity selectors, SCIM must be configured first. Without SCIM, only email/email domain/service token selectors work.

8. **Split tunnel mode is a global decision** — Changing from Exclude to Include mode mid-deployment is disruptive. Decide early and document clearly.

9. **Browser Rendering vs. Browser Isolation** — Browser Rendering (SSH/VNC in browser) is an Access app feature. Browser Isolation (BISO) is a separate service that renders ALL web content remotely. Don't confuse them.

10. **Current-state diagrams should be simple** — Aim for ≤25 elements. Show topology structure, not every policy or rule. Use vendor-specific types (`zscaler_cloud`, `vendor_cloud`) not CF1 types.

## Related Skills

### Implementation Skills (in recommended order)
- **access-identity**: Identity foundation — IdP + SCIM
- **tunnel-deployment**: CF Tunnel connectivity to private networks
- **tunnel-routing**: CIDR and hostname routes on tunnels
- **gateway-policy-design**: Internet security policies (DNS, HTTP, Network)
- **gateway-tls-inspection**: TLS decryption for HTTP inspection
- **gateway-dns-security**: DNS-layer protection
- **gateway-network-policies**: Network-level filtering
- **access-app-setup**: Access applications for private/public/SaaS resources
- **access-private-network**: Private network access patterns
- **access-saas-federation**: SaaS SSO federation
- **device-posture-setup**: Endpoint compliance checks
- **casb-integration**: SaaS posture scanning (API CASB)
- **gateway-dlp**: Data loss prevention
- **wan-site-connectivity**: Site-to-site via CF WAN

### Migration Skills
- **migrate-zscaler-zia**: Zscaler ZIA → CF1 Gateway
- **migrate-zscaler-zpa**: Zscaler ZPA → CF1 Access + Tunnel
- **migrate-palo-alto**: Palo Alto → CF1 Gateway + Access + Posture
