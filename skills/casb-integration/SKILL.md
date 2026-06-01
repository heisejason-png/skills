---
name: "casb-integration"
description: "Set up CASB SaaS integrations for API-based posture scanning. Covers integration setup for Google Workspace, M365, Dropbox, GitHub, Slack, and other SaaS applications, including vendor-specific authorization requirements, scan policies, and DLP enablement."
domain: "casb"
stage: ["assess", "design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/applications/scan-apps/"
---

# CASB SaaS Integration Setup

## Persona

You are a senior Cloudflare One architect specializing in SaaS security posture management. You design CASB integrations that provide visibility into SaaS misconfigurations, data exposure, and user behavior across vendors. You understand the authorization requirements for each vendor cold, and you know the difference between API-based scanning and inline CASB controls.

## When to Use

- Customer wants to scan SaaS applications for security misconfigurations
- Customer needs visibility into file sharing, user permissions, or admin settings in SaaS apps
- Customer asks about API CASB vs inline CASB
- Customer wants to onboard a new SaaS vendor (Google Workspace, Microsoft 365, Dropbox, GitHub, Slack, Salesforce, Box, etc.)
- Customer asks "what does CASB scan?" or "what SaaS apps can I connect?"
- Customer wants DLP scanning of data at rest in SaaS apps (files, messages, etc.)

## When NOT to Use

- Customer needs inline DLP scanning of SaaS uploads/downloads in real-time → use `gateway-dlp`
- Customer needs to triage existing CASB findings → use `casb-posture-triage`
- Customer wants SSO/identity federation for a SaaS app → use `access-saas-federation`
- Customer wants to restrict which SaaS tenants users can access (tenant control) → use `gateway-policy-design` with HTTP policies

## Assess

Before setting up a CASB integration, gather:

1. **Which SaaS applications?**
   - Google Workspace, Microsoft 365, Dropbox, GitHub, Slack, Salesforce, Box, Jira, ServiceNow, Zoom, AWS, GCP, Okta, Confluence, Bitbucket, Workday?
   - Each vendor has specific authorization requirements and scanned asset types.

2. **What admin access is available?**
   - Google Workspace: Requires a **Super Admin** account. Must configure domain-wide delegation with specific OAuth scopes in the Google Admin Console.
   - Microsoft 365: Requires a **Global Administrator** to grant admin consent for the Cloudflare application in Azure AD.
   - Dropbox: Requires a **Team Admin** account for OAuth.
   - GitHub: Requires an **Organization Owner** account for OAuth.
   - AWS: Requires IAM credentials with specific read-only permissions (60+ permissions across EC2, S3, IAM, GuardDuty, etc.).
   - GCP: Requires a service account with `cloud-platform.read-only` and storage/compute read permissions.

3. **What scan policy is needed?**
   - **Standard policy**: Read-only scanning of misconfigurations, user settings, file permissions.
   - **DLP-enabled policy**: Adds data-at-rest scanning of file contents and messages for sensitive data patterns. Available for Microsoft 365 and Google Workspace. Requires additional permissions (e.g., `Mail.ReadWrite`, `Files.Read.All`).

4. **What's the org size?**
   - Large orgs (10K+ users) will generate significant scan volume. Initial onboarding creates crawl tasks for every user across multiple crawlers. Plan for the first scan cycle to take 24-48 hours.
   - Very large orgs (50K+ users) may cause temporary load spikes during initial onboarding.

5. **Compliance requirements?**
   - Data residency constraints that affect which SaaS data can be accessed?
   - Audit logging requirements for CASB scan activity?

## Design

### API CASB vs Inline CASB

This is the key architectural decision. They are complementary, not competing.

| Capability | API CASB (This Skill) | Inline CASB (Gateway HTTP Policies) |
|---|---|---|
| **How it works** | Out-of-band API scanning of SaaS tenant data | Real-time inspection of HTTP traffic through Gateway |
| **What it finds** | Misconfigurations, overshared files, weak auth settings, risky users | Sensitive data in uploads/downloads, unauthorized SaaS usage |
| **When it runs** | Periodic background scans (configurable frequency) | Every HTTP request through Gateway |
| **Requires** | OAuth/API credentials to the SaaS tenant | CF1 Client (WARP) deployed, Gateway HTTP policies configured |
| **Changes data?** | No — read-only scanning | Can block/allow/isolate in real-time |
| **Best for** | Posture assessment, compliance, shadow IT discovery, data exposure audit | Real-time DLP enforcement, tenant control, action blocking |

**Recommendation**: Start with API CASB for visibility, then layer inline controls via Gateway for enforcement. API CASB shows you what's wrong; Gateway policies prevent it from getting worse.

### Vendor Coverage Decision Matrix

| Vendor | Asset Types Scanned | DLP Support | Auth Method |
|---|---|---|---|
| **Google Workspace** | Users, Files, File Permissions, Drives, Groups, Calendars, Mail Settings, Labels | Yes | Domain-wide delegation (service account + OAuth scopes) |
| **Microsoft 365** | Users, Files, Folders, Drives, Groups, Roles, Apps, Domains, Calendars, Mail Rules, Auth Methods, Risky Users, Labels, Copilot activity | Yes | Azure AD admin consent (app registration) |
| **Dropbox** | Users, Files, File Permissions, Folders, Groups | No | OAuth (Team Admin) |
| **GitHub** | Users, Repositories, Commits, Issues, Packages, Webhooks, Submodules, Organizations | No | OAuth (Org Owner) |
| **Slack** | Users, Channels, Messages, Workspaces | No | OAuth (Workspace Admin) |
| **AWS** | Users, Roles, Buckets, Bucket Permissions, Instances, Security Groups, Certificates, Servers | No | IAM credentials (access key + secret) |
| **GCP** | Projects, Buckets, Bucket IAM Permissions, Instances | No | Service account key |
| **Salesforce** | Users, Roles, Groups, Reports, Sites | No | OAuth (System Admin) |
| **Jira** | Users, Projects, Issues | No | OAuth (Admin) |

### Scan Policy Selection

Each integration is connected with a **scan policy** that determines permissions:

- **Standard Policy**: Read-only access to configuration and metadata. Discovers misconfigurations, permission issues, and user security settings.
- **DLP-Enabled Policy** (Google Workspace, Microsoft 365 only): Adds content inspection. Scans file contents, email bodies, and messages for sensitive data matching DLP profiles. Requires broader permissions (file read, mail read).

**Recommendation**: Start with the Standard Policy. Enable DLP only when you have DLP profiles configured and a clear use case for data-at-rest scanning. DLP-enabled scans are more resource-intensive and require broader permissions that may need additional approval from the SaaS tenant admin.

## Implement

### With MCP Tools (Code Mode)

> Note: CASB endpoints are not discoverable via `search`. Use `execute` with the API paths below directly.

**Step 1: Check existing integrations**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/casb/integrations` })
  — review existing integrations to avoid duplicates. Note vendor, status, and credential expiry.
```

**Step 2: Create the integration via Dashboard** (integration creation requires OAuth authorization flow)

Integration creation requires OAuth authorization flow which must happen in the Dashboard. See the "Without MCP Tools" section below for vendor-specific setup steps.

**Step 3: Verify the integration was created**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/casb/integrations` })
  — confirm the new integration appears with status "Healthy"
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/casb/integrations/${integrationId}` })
  — verify permissions, credentials_expiry, and last_hydrated
```

**Step 4: Monitor initial scan progress**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/gateway/casb/integrations/${integrationId}/assets` })
  — check that assets are being discovered. First scan may take 24-48 hours for large orgs.
```

### Without MCP Tools (Dashboard)

**General flow for all vendors:**
1. **Dashboard**: Zero Trust → CASB → Integrations → Add integration
2. Select the SaaS vendor
3. Choose the scan policy (Standard or DLP-enabled if available)
4. Complete the vendor-specific authorization flow (see below)
5. Name the integration and save

**Google Workspace:**
1. You'll need to configure **domain-wide delegation** in the Google Admin Console:
   - Go to Google Admin Console → Security → API Controls → Domain-wide Delegation
   - Add the Cloudflare service account client ID
   - Grant the OAuth scopes shown during integration setup (calendar, drive, directory, gmail, etc.)
2. Provide the **administrator email** (Super Admin account) in the CF1 dashboard
3. Save — CF1 will use the service account with domain-wide delegation to scan

**Microsoft 365:**
1. Click the authorization link — redirects to Microsoft login
2. Sign in as a **Global Administrator**
3. Grant admin consent for the Cloudflare application permissions
4. Redirect back to CF1 dashboard — integration is created automatically

**AWS:**
1. Create an IAM policy in AWS with the required read-only permissions
2. Create an IAM user or role with that policy attached
3. Generate an access key and secret key
4. Enter the credentials in the CF1 dashboard

**GitHub:**
1. Click the authorization link — redirects to GitHub
2. Sign in as an **Organization Owner**
3. Grant access to the target organization(s)
4. Redirect back to CF1 dashboard

### Without MCP Tools (API)

```bash
# List existing integrations
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/casb/integrations" \
  -H "Authorization: Bearer {api_token}" | jq '.result'

# Get a specific integration
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/gateway/casb/integrations/{integration_id}" \
  -H "Authorization: Bearer {api_token}" | jq '.result'
```

Note: Integration creation is done through the Dashboard OAuth flow. The API is primarily used for listing and inspecting integrations.

## Validate

1. **Check integration health:**
   - In the Dashboard: Zero Trust → CASB → Integrations — status should show **Healthy** (green)
   - With MCP: GET `/accounts/${accountId}/gateway/casb/integrations/${integrationId}` — verify `status: "Healthy"`, `credentials_expiry` is in the future

2. **Verify asset discovery:**
   - Wait 1-4 hours after creation for initial asset discovery
   - With MCP: GET `/accounts/${accountId}/gateway/casb/integrations/${integrationId}/assets` — confirm assets are appearing
   - In Dashboard: Zero Trust → CASB → Findings — should show findings grouped by severity

3. **Verify posture findings:**
   - Wait 4-24 hours for first findings to appear (depends on org size)
   - In Dashboard: Zero Trust → CASB → Findings — search for common findings like "mfa", "sharing", "public", "admin"

4. **Check edge cases:**
   - If DLP-enabled: Verify DLP profiles exist (GET `/accounts/${accountId}/dlp/profiles`) — findings won't appear without active profiles
   - For Google Workspace: Verify domain-wide delegation scopes match the scan policy — missing scopes will cause partial scan failures
   - For Microsoft 365: Verify admin consent was granted for all requested permissions in Azure AD → Enterprise Applications

5. **Validate credential lifecycle:**
   - Check `credentials_expiry` — OAuth tokens expire and must be refreshed
   - Healthy integrations auto-refresh credentials. If status is "Unhealthy", credentials may have expired or been revoked

## Operate

### Monitoring

- **Integration health**: Check integration status regularly. Unhealthy integrations stop producing findings.
- **Credential expiry**: Monitor `credentials_expiry` field. Some vendors (Jira, certain OAuth flows) don't auto-refresh.
- **Finding counts**: A sudden drop in findings may indicate a broken integration, not improved security posture.
- **Finding spikes**: A sudden spike in findings usually indicates a code fix that unblocked previously stuck scans, or a new integration completing its first full scan cycle. Investigate before escalating.
- **Asset counts**: Monitor asset discovery over time. Decreasing asset counts may indicate permission changes in the SaaS tenant.

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Integration status "Unhealthy" | OAuth token expired or revoked, or SaaS admin removed permissions | Re-authorize the integration: Dashboard → CASB → Integrations → click the integration → Re-authorize |
| No findings after 24 hours | Integration may be scanning but hasn't completed first cycle; or no misconfigurations found (unlikely) | Check `last_hydrated` timestamp via GET `/accounts/${accountId}/gateway/casb/integrations/${integrationId}`. If it's recent, scans are running. If stale, check health. |
| Missing asset types | Scan policy doesn't include required permissions for that asset type | Check policy permissions vs vendor documentation. May need to re-create with a broader policy. |
| DLP findings not appearing | DLP profiles not configured, or DLP not enabled on the scan policy | Verify DLP profiles exist (GET `/accounts/${accountId}/dlp/profiles`). Verify integration policy has `dlp_enabled: true`. |
| Google Workspace: partial scan failures | Domain-wide delegation scopes missing or incorrect | Verify all required scopes are configured in Google Admin Console → Security → API Controls → Domain-wide Delegation |
| Microsoft 365: "Insufficient privileges" | Admin consent not fully granted, or permissions were modified in Azure AD | Re-grant admin consent: Azure AD → Enterprise Applications → Cloudflare → Permissions → Grant admin consent |
| Findings spike after integration update | Code fix or scan improvement unblocked previously stuck scans — not a misconfiguration issue | Investigate the timing relative to CASB release notes. New findings are legitimate but may not represent new risk. |
| Large org onboarding slow | First scan creates crawl tasks per-user across all crawlers — can take 24-48 hours for orgs with 10K+ users | This is expected. Monitor asset counts over time. Don't re-create the integration. |

### When to Revisit

- SaaS admin changes permissions or revokes OAuth → re-authorize
- New SaaS applications adopted by the org → add new integrations
- Compliance audit requires data-at-rest scanning → upgrade to DLP-enabled policy
- Org grows significantly → monitor for scan performance and finding volume
- Vendor adds new security features (e.g., Microsoft Copilot) → check for new asset categories

## Gotchas

1. **Google Workspace domain-wide delegation is the #1 setup failure** — The OAuth scopes must be configured in the Google Admin Console, not just in CF1. If any scope is missing, the corresponding asset type won't be scanned. The administrator email must be a Super Admin. This is a multi-step process that customers frequently get wrong.

2. **DLP requires both an enabled scan policy AND configured DLP profiles** — Enabling DLP on the integration alone does nothing. You also need DLP profiles with detection entries. If GET `/accounts/${accountId}/dlp/profiles` returns empty, DLP findings will never appear.

3. **Large org onboarding creates massive initial scan load** — A 50K+ user Microsoft org will generate hundreds of thousands of crawl tasks across multiple crawlers on first connect. This is normal but can take 24-48 hours to complete. Do not re-create the integration if it appears slow — this will restart the process.

4. **Integration credentials expire silently** — OAuth tokens have expiry dates. Most vendors auto-refresh, but some (Jira, older OAuth flows) may not. Always check `credentials_expiry` when investigating stale findings. An "Unhealthy" status usually means credential refresh failed.

5. **Findings are not real-time** — API CASB scans run on a periodic schedule (typically every few hours per crawler). A file shared publicly right now won't generate a finding for hours. For real-time protection, layer Gateway HTTP policies with DLP and tenant control.

6. **Re-authorizing an integration resets scan state** — When you re-authorize a broken integration, it may trigger a full re-scan. For large orgs, this means another 24-48 hour initial scan cycle. Only re-authorize when necessary.

7. **Asset categories vary significantly by vendor** — Microsoft has 21+ asset category types (including Copilot activity). Google Workspace has ~12. Dropbox has ~5. Set expectations with the customer about what visibility each vendor provides.

8. **Unhealthy ≠ broken forever** — An integration can go Unhealthy temporarily due to transient API errors on the SaaS vendor side. Check again after a few hours before re-authorizing. The `last_hydrated` timestamp shows when the last successful credential refresh occurred.

## Related Skills

- **casb-posture-triage**: Triage and remediate findings after integration is set up. This is the natural next step.
- **gateway-dlp**: Configure DLP profiles that API CASB uses for data-at-rest scanning. Also configures inline DLP enforcement.
- **dlp-profile-design**: Design DLP detection profiles (predefined + custom). Required before enabling DLP on CASB integrations.
- **access-saas-federation**: SSO federation for the same SaaS apps — complementary but separate from CASB scanning.
- **risk-scoring**: CASB findings and user behaviors feed into risk scores. Chain to this for user-level risk assessment.
