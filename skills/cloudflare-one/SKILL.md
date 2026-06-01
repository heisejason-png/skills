---
name: cloudflare-one
description: "Guides Cloudflare One Zero Trust and SASE work across Access, Gateway, WARP, Tunnel, Magic WAN, DLP, CASB, device posture, and identity. Use when designing, configuring, troubleshooting, or reviewing Cloudflare One deployments. Retrieval-first: use current Cloudflare docs/API schemas instead of embedded product docs."
---

# Cloudflare One

Do not use this skill as product documentation. Before citing limits, settings, API fields, category IDs, or exact UI paths, retrieve current information from `developers.cloudflare.com/cloudflare-one/`, the Cloudflare docs MCP server, or the Cloudflare API schema.

## Workflow

1. Classify the ask: architecture, configuration, troubleshooting, migration, or review.
2. Gather context: account ID, users/sites/apps, identity provider, SCIM/group sync, device management, traffic path, compliance constraints, and rollout blast radius.
3. Retrieve only the current docs needed for the products involved: Access, Gateway, WARP, Tunnel, Magic WAN, DLP, CASB, device posture, or identity.
4. If account access is available, inspect existing resources before proposing or making changes: Access apps/policies/groups/IdPs, Gateway rules/lists/categories, device profiles/posture checks, tunnels/routes, DNS/resolver settings, and locations/sites.
5. Propose the change set with prerequisites, validation, and rollback. For risky changes, stage disabled or scoped to a pilot group/site unless the user explicitly asks otherwise.

## Docs to Retrieve

- [Cloudflare One](https://developers.cloudflare.com/cloudflare-one/)
- [Choose an Access application type](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/choose-application-type/)
- [Access policies](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/)
- [Gateway traffic policies](https://developers.cloudflare.com/cloudflare-one/traffic-policies/)
- [Gateway identity selectors](https://developers.cloudflare.com/cloudflare-one/traffic-policies/identity-selectors/)
- [SCIM provisioning](https://developers.cloudflare.com/cloudflare-one/team-and-resources/users/scim/)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
- [Tunnel connectivity prechecks](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/troubleshoot-tunnels/connectivity-prechecks/)
- [Split Tunnels](https://developers.cloudflare.com/cloudflare-one/team-and-resources/devices/cloudflare-one-client/configure/route-traffic/split-tunnels/)
- [Virtual networks](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/cloudflared/tunnel-virtual-networks/)
- [Resolver policies](https://developers.cloudflare.com/cloudflare-one/traffic-policies/resolver-policies/)
- [HTTP policies](https://developers.cloudflare.com/cloudflare-one/traffic-policies/http-policies/)
- [TLS decryption](https://developers.cloudflare.com/cloudflare-one/traffic-policies/http-policies/tls-decryption/)
- [Data Loss Prevention](https://developers.cloudflare.com/cloudflare-one/data-loss-prevention/)
- [CASB findings](https://developers.cloudflare.com/cloudflare-one/cloud-and-saas-findings/manage-findings/)
- [User risk score](https://developers.cloudflare.com/cloudflare-one/team-and-resources/users/risk-score/)
- [Cloudflare WAN](https://developers.cloudflare.com/cloudflare-wan/)
- [Cloudflare Network Firewall](https://developers.cloudflare.com/cloudflare-network-firewall/)

## Assessment Prompts

Use these to avoid jumping straight to configuration. Ask only the prompts relevant to the user's task.

### Architecture and Current State

- Sites and users: offices, branches, data centers, VPCs, remote users, contractors, user counts, and current connectivity model.
- Applications and destinations: SaaS, public apps, private apps, APIs, infrastructure targets, protocols, ports, hostnames, and IP ranges.
- Connectivity: VPN, MPLS, SD-WAN, direct Internet breakout, centralized backhaul, site-to-site needs, and private DNS architecture.
- Security stack: current SWG, NGFW, VPN/ZTNA, DLP, CASB, email security, logging, and compliance requirements.
- Identity: IdP, SCIM/group sync, group naming, multi-IdP needs, service accounts, and contractor/partner access.
- Rollout: pilot users/sites, blast radius, rollback path, support owners, and success criteria.

### Access and SaaS Federation

- App shape: web app, API, SSH/RDP/VNC, database, SaaS app, public hostname, private IP, or private hostname.
- Access model: clientless browser access, WARP private access, service token access, or SaaS SSO federation.
- Policy needs: user groups, device posture, geography, session duration, mTLS, service tokens, and app launcher visibility.
- SaaS details: SAML vs OIDC support, ACS/redirect URLs, Entity IDs/client IDs, required attributes, and tenant-control requirements.

### Tunnel and Private Networking

- Sites and segments: which data centers, VPCs, offices, or network segments need connectivity.
- HA: dev/test single connector, production multiple connectors, or advanced multi-tunnel/site redundancy.
- Runtime: where cloudflared or WARP Connector will run: VM, container, Kubernetes, bare metal, or other target.
- Egress: whether connectors can reach Cloudflare over the required outbound ports/protocols. Retrieve current Tunnel docs before naming exact endpoints.
- Origin reachability: whether the connector can resolve and reach every private origin.
- Routing: required CIDRs/hostnames, overlapping IP spaces, virtual networks, split tunnel mode, and private DNS/resolver policy needs.
- Management model: prefer remotely managed/token-based tunnels for new deployments unless there is a clear reason for local config.

### Gateway, TLS, and DLP

- Traffic controls: DNS categories, HTTP URL/path inspection, L4 ports/protocols, egress IP requirements, custom lists, and allow/block exceptions.
- Identity: whether Gateway policies need user or group selectors, and whether users will be authenticated through WARP/IdP context.
- TLS inspection: root CA deployment path, certificate-pinned applications, compliance exceptions, and FIPS requirements.
- DLP: sensitive data types, channels to inspect, TLS inspection readiness, DLP profiles, payload logging requirements, and false-positive tolerance.

### CASB, Device Posture, and Risk

- CASB: SaaS vendors, admin access level, scan policy, org size, remediation owner, and whether inline protection is also required.
- Device posture: required checks, third-party EDR/MDM integrations, enrollment rules, device profiles, and split tunnel alignment.
- Risk scoring: relevant behavior signals, false-positive sources such as VPNs or service accounts, and whether risk is for investigation or enforcement.

### Magic WAN / Site Connectivity

- Site topology, on-ramp type, route ownership, tunnel redundancy, static vs BGP-managed routes, network firewall needs, and appliance/profile ownership.

## High-Value Guardrails

- Access controls application authorization; Gateway controls traffic inspection/filtering. Use both when the requirement spans identity-aware app access and network/web security.
- Public hostname Access apps can be clientless. Private destination apps require WARP or another network on-ramp plus routes and DNS resolution.
- Cloudflare Tunnel is an off-ramp from a private network to Cloudflare. Magic WAN/site connectivity is not a drop-in replacement for per-user application access.
- Group-based policies depend on IdP group claims or SCIM. If group sync is missing, do not invent group selectors.
- Private hostnames need explicit DNS routing/resolution; creating an Access app alone is not enough.
- HTTP inspection and DLP for encrypted web traffic require TLS inspection and planned Do Not Inspect exceptions.
- Gateway DNS, Network, HTTP, and Egress policies have different evaluation semantics. Retrieve current docs before explaining precedence.
- Start broad block/allow/DLP/TLS policies disabled, in audit, or limited to a pilot unless the user approves a wider rollout.

## Documented Guardrails

These points are documented in Cloudflare docs, but are easy to miss when composing multiple Cloudflare One products. Treat them as checks, not as the source of truth.

### Identity and Access

- Access Groups are Cloudflare objects; IdP/SCIM groups are identity claims. Gateway group selectors use synced IdP groups, not Access Groups.
- Group names and SAML/OIDC attributes are case-sensitive. Verify exact claim names and values before creating group-based rules.
- SCIM changes and group membership can be stale until sync and re-authentication complete. Troubleshoot with the user's last authenticated identity, not just the IdP state.
- Access policies are default-deny. A private app with routes but no Allow policy still blocks access.
- Access policy selectors can use IP lists, not Gateway domain or URL lists.
- SaaS federation handles authentication into the SaaS app. SaaS authorization and tenant restrictions usually require SaaS-side roles and/or Gateway tenant controls.
- Browser Rendering for SSH/VNC/RDP is an Access capability. Browser Isolation renders general web content remotely. Do not conflate them.

### Private Networking

- Split tunnel mode changes the meaning of every route decision: Exclude mode sends traffic to Cloudflare when removed from excludes; Include mode sends traffic only when added to includes.
- Virtual networks must be assigned consistently to tunnel routes and the WARP device profile. One side alone creates unreachable routes.
- CIDR routes and hostname routes solve different problems. Hostname routes still need DNS resolution through resolver policies or Local Domain Fallback.
- A healthy tunnel only proves cloudflared can reach Cloudflare. The connector must also resolve and reach the private origin.
- Run multiple cloudflared connectors for production HA, preferably on separate hosts. Token-based, remotely managed tunnels are the default for new deployments.

### Gateway, TLS, and DLP

- `dns.domains` matches a domain and subdomains; `dns.fqdn` is exact-match only.
- DNS pre-resolution selectors and post-resolution selectors do not behave like a single strict precedence list. Retrieve current evaluation docs before changing rule order.
- HTTP Do Not Inspect rules run before HTTP Allow/Block/Isolate behavior. A later block rule will not override an earlier inspection bypass.
- Certificate-pinned apps need Do Not Inspect exceptions before broad TLS inspection. Deploy the Cloudflare root CA to managed devices before enabling inspection.
- DLP profiles are detection definitions only. They do nothing until referenced by Gateway HTTP policies or CASB scan settings.
- Start DLP with monitoring/payload logging where appropriate, tune false positives, then block.
- Gateway Network policies are strict L4 controls. Identity-aware L4 matching requires authenticated WARP context.

### CASB, Risk, and Operations

- API CASB is out-of-band and periodic. It does not provide real-time inline enforcement; use Gateway/DLP/tenant controls for inline protection.
- CASB findings are tied to specific assets and instances. Drill into affected assets before recommending remediation.
- Use current Dashboard remediation guidance for CASB fixes. Most remediations happen in the SaaS admin console, not Cloudflare.
- Large SaaS integrations can take 24-48 hours for initial scans. Reauthorizing can restart scan state; check credential health before reconnecting.
- User risk scores are behavior-based and asynchronous. CASB findings do not automatically imply high user risk.

### Magic WAN / Site Connectivity

- Magic WAN is connectivity, not a security service. Apply inspection and policy with Gateway/network firewall where required.
- WAN firewall expressions are not the same language as Gateway wirefilter expressions. Retrieve the current syntax before editing.
- Generated IPsec PSKs and some OAuth/client secrets are returned once. Store them immediately.

## Output Defaults

- Designs: current assumptions, target architecture, product responsibilities, rollout phases, validation, and open decisions.
- Configuration work: prerequisites, exact resources to inspect/create/change, test cases, and rollback.
- Troubleshooting: traffic path, likely failure point, evidence to collect, and next test.

## Validation Prompts

- Access: test authorized, unauthorized, posture-failing, service-token, and multi-IdP flows when applicable; inspect logs and policy precedence.
- Private network access: verify route lookup, tunnel health, origin reachability, split tunnel behavior, DNS resolution, and end-to-end access from a WARP test device.
- Gateway: verify rule type, action, traffic expression, precedence/evaluation phase, referenced lists, and Gateway settings before enabling broadly.
- TLS/DLP: test Do Not Inspect exceptions and root CA trust before enabling inspection; test DLP with known samples and monitor false positives before blocking.
- CASB/risk: confirm integration health, credential expiry, asset discovery, scan timing, finding instances, and risk-score signal latency before declaring remediation complete.
- Magic WAN: verify tunnel health, route priority/ownership, traffic flow, firewall expression syntax, and connector/appliance telemetry where applicable.

## API Safety

- Use fully qualified MCP tool names when MCP tools are available.
- Never guess category IDs, application IDs, wirefilter fields, or API request bodies. Retrieve the current schema/docs and existing account objects.
- Do not enable broad production policies without explicit approval.
