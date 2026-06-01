# Cloudflare Skills

A collection of [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) for building on Cloudflare, Workers, the Agents SDK, and the wider Cloudflare Developer Platform.

## Installing

These skills work with any agent that supports the Agent Skills standard, including Claude Code, OpenCode, OpenAI Codex, and Pi.

### Claude Code

Install using the [plugin marketplace](https://code.claude.com/docs/en/discover-plugins#add-from-github):

```
/plugin marketplace add cloudflare/skills
/plugin install cloudflare@cloudflare
```

### Cursor

Install from the Cursor Marketplace or add manually via **Settings > Rules > Add Rule > Remote Rule (Github)** with `cloudflare/skills`.

### npx skills

Install using the [`npx skills`](https://skills.sh) CLI:

```
npx skills add https://github.com/cloudflare/skills
```

### Clone / Copy

Clone this repo and copy the skill folders into the appropriate directory for your agent:

| Agent | Skill Directory | Docs |
|-------|-----------------|------|
| Claude Code | `~/.claude/skills/` | [docs](https://code.claude.com/docs/en/skills) |
| Cursor | `~/.cursor/skills/` | [docs](https://cursor.com/docs/context/skills) |
| OpenCode | `~/.config/opencode/skills/` | [docs](https://opencode.ai/docs/skills/) |
| OpenAI Codex | `~/.codex/skills/` | [docs](https://developers.openai.com/codex/skills/) |
| Pi | `~/.pi/agent/skills/` | [docs](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent#skills) |

## Commands

Commands are user-invocable slash commands that you explicitly call.

| Command | Description |
|---------|-------------|
| `/cloudflare:build-agent` | Build an AI agent on Cloudflare using the Agents SDK |
| `/cloudflare:build-mcp` | Build an MCP server on Cloudflare |

## Skills

Skills are contextual and auto-loaded based on your conversation. When a request matches a skill's triggers, the agent loads and applies the relevant skill to provide accurate, up-to-date guidance.

| Skill | Useful for |
|-------|------------|
| cloudflare | Comprehensive platform skill covering Workers, Pages, storage (KV, D1, R2), AI (Workers AI, Vectorize, Agents SDK), networking (Tunnel, Spectrum), security (WAF, DDoS), and IaC (Terraform, Pulumi) |
| agents-sdk | Building stateful AI agents with state, scheduling, RPC, MCP servers, email, and streaming chat |
| durable-objects | Stateful coordination (chat rooms, games, booking), RPC, SQLite, alarms, WebSockets |
| sandbox-sdk | Secure code execution for AI code execution, code interpreters, CI/CD systems, and interactive dev environments |
| wrangler | Deploying and managing Workers, KV, R2, D1, Vectorize, Queues, Workflows |
| web-perf | Auditing Core Web Vitals (FCP, LCP, TBT, CLS), render-blocking resources, network chains |
| building-mcp-server-on-cloudflare | Building remote MCP servers with tools, OAuth, and deployment |
| building-ai-agent-on-cloudflare | Building AI agents with state, WebSockets, and tool integration |

## Cloudflare One (CF1 Stack)

Skills for deploying, migrating to, and operating [Cloudflare One](https://developers.cloudflare.com/cloudflare-one/) — Cloudflare's SASE and zero trust platform. Each skill encodes deployment knowledge, operational judgment, and architectural patterns. Each skill works standalone as authoritative guidance and integrates directly with [Cloudflare's MCP server](https://github.com/cloudflare/mcp).

| Skill | Useful for |
|-------|------------|
| access-app-setup | Creating Access applications (self-hosted, private, SaaS, infrastructure) with policy design |
| access-identity | Identity provider setup, SCIM provisioning, group mapping, multi-IdP strategies |
| access-private-network | Private network access patterns with WARP, tunnels, split tunnels, virtual networks |
| access-saas-federation | SaaS SSO federation through Access (SAML/OIDC), app launcher, tenant control |
| architecture-design | Environment assessment, CF1 architecture proposals, current/future state design |
| casb-integration | SaaS integration setup for API CASB scanning, vendor-specific authorization |
| casb-posture-triage | Triaging CASB posture findings, severity assessment, remediation prioritization |
| device-posture-setup | Device posture rules: OS version, disk encryption, firewall, EDR integrations |
| dlp-profile-design | DLP detection profiles, custom regex, exact-match datasets, compliance templates |
| gateway-dlp | DLP profile integration with Gateway HTTP policies for inline data protection |
| gateway-dns-security | DNS filtering, content categories, custom blocklists, safe search enforcement |
| gateway-network-policies | Layer 4 network policies for port/protocol filtering and private network controls |
| gateway-policy-design | Gateway policy architecture (DNS, HTTP, Network, Egress), evaluation order, wirefilter |
| gateway-tls-inspection | TLS decryption setup, certificate deployment, Do Not Inspect bypass rules |
| migrate-palo-alto | Migrating from Palo Alto (PAN-OS, Prisma Access, GlobalProtect) to Cloudflare One |
| migrate-zscaler-zia | Migrating from Zscaler ZIA to CF1 Gateway (URL filtering, firewall, DLP, identity) |
| migrate-zscaler-zpa | Migrating from Zscaler ZPA to CF1 Access + Tunnels (app segments, connectors, policies) |
| risk-scoring | User risk scoring, behavior configuration, risk-based policy integration |
| tunnel-deployment | Deploying Cloudflare Tunnels, HA patterns, cloudflared setup, health checks |
| tunnel-routing | CIDR routes, hostname routes, virtual networks, resolver policies for private DNS |
| wan-site-connectivity | Site-to-site WAN connectivity via IPsec/GRE tunnels, static routes, network firewall |

## MCP Servers

This plugin includes [Cloudflare's remote MCP servers](https://developers.cloudflare.com/agents/model-context-protocol/mcp-servers-for-cloudflare/) for enhanced functionality:

| Server | Purpose |
|--------|---------|
| cloudflare-api | Manage Cloudflare account resources, zones, and settings |
| cloudflare-docs | Up-to-date Cloudflare documentation and reference |
| cloudflare-bindings | Build Workers applications with storage, AI, and compute primitives |
| cloudflare-builds | Manage and get insights into Workers builds |
| cloudflare-observability | Debug and analyze application logs and analytics |

## Resources

- [Cloudflare Agents Documentation](https://developers.cloudflare.com/agents/)
- [Cloudflare MCP Guide](https://developers.cloudflare.com/agents/model-context-protocol/)
- [Agents SDK Repository](https://github.com/cloudflare/agents)
- [Agents Starter Template](https://github.com/cloudflare/agents-starter)
