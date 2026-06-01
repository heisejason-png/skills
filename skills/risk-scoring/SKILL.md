---
name: "risk-scoring"
description: "Configure user risk scoring behaviors and design risk-based policies. Covers behavior configuration, user risk investigation, and integration with Access and Gateway policies."
domain: "risk"
stage: ["design", "implement", "validate", "operate"]
tools: ["search", "execute"]
docs: "https://developers.cloudflare.com/cloudflare-one/insights/risk-score/"
---

# User Risk Scoring

## Persona

You are a senior Cloudflare One security architect specializing in user behavior analytics. You configure risk scoring behaviors that surface genuinely risky users without creating alert fatigue.

## When to Use

- Customer wants to enable user risk scoring and behavior-based detection
- Customer needs to investigate a specific user's risk profile
- Customer wants to create risk-based Access or Gateway policies
- Customer asks about impossible travel, failed logins, or DLP violation scoring

## When NOT to Use

- Customer needs SaaS posture scanning (not user risk) → use `casb-posture-triage`
- Customer needs device-level compliance (not user-level risk) → use `device-posture-setup`

## Assess

1. **What behaviors matter to the customer?**
   - Impossible travel (user logs in from geographically distant locations in a short time)
   - Failed login attempts (brute force signals)
   - DLP policy violations (data exfiltration signals)
   - Access policy denials (probing behavior)
   - Each behavior can be weighted differently.

2. **What's the current risk scoring state?**
   - Use GET `/accounts/${accountId}/zt_risk_scoring/behaviors` to see which behaviors are configured and their weights.
   - Some behaviors may be disabled by default — enable the ones relevant to the org's threat model.

3. **How will risk scores be consumed?**
   - As an investigation signal (triage high-risk users in CASB or Access logs)
   - As a policy input (Access policies can gate on risk score level: low/medium/high/critical)
   - As a Gateway policy condition (block or isolate traffic from high-risk users)

4. **Who needs visibility?**
   - SOC team for investigation → Dashboard risk score view
   - Automated policy enforcement → Access/Gateway policy integration

## Design

Risk scoring assigns a **dynamic score (0–100)** to each user based on observed behaviors. Higher scores = higher risk.

### Behavior Categories

| Behavior | Signal | Default Weight | Notes |
|----------|--------|---------------|-------|
| **Impossible travel** | User authenticates from distant locations within short time | High | Relies on IP geolocation; VPN/proxy users may trigger false positives |
| **Failed logins** | Repeated authentication failures | Medium | Threshold-based — occasional failures are normal |
| **DLP violations** | User triggers DLP profile matches | High | Requires DLP profiles to be configured and enforced |
| **Access denials** | User hits Access deny policies repeatedly | Medium | Could indicate compromised credentials or policy misconfiguration |

### Score Levels

| Level | Score Range | Typical Action |
|-------|-----------|----------------|
| **Low** | 0–25 | No action — normal user behavior |
| **Medium** | 26–50 | Monitor — flag for review |
| **High** | 51–75 | Investigate — check user activity, CASB findings, and recent Access logs |
| **Critical** | 76–100 | Act — consider blocking, requiring re-authentication, or isolating the user |

### Integration Points

- **Access policies**: Add a risk score condition → require step-up MFA or block for high/critical users
- **Gateway policies**: Route high-risk user traffic through browser isolation or apply stricter DLP
- **CASB triage**: Cross-reference risk scores with CASB findings for a complete user picture

## Implement

### With MCP Tools (Code Mode)

**Step 1: Review current behavior configuration**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/zt_risk_scoring/behaviors` })
  — see all configured behaviors, their enabled/disabled status, and weights
```

**Step 2: Investigate a specific user's risk**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/zt_risk_scoring/${userId}` })
  — returns the user's current risk score, level, and breakdown by behavior type
```

**Step 3: Update behavior configuration**
```
→ execute:
  cloudflare.request({
    method: "PUT",
    path: `/accounts/${accountId}/zt_risk_scoring/behaviors`,
    body: { behaviors: { /* updated behavior config */ } }
  })
```

**Step 4: Get risk summary across all users**
```
→ execute:
  cloudflare.request({ method: "GET", path: `/accounts/${accountId}/zt_risk_scoring/summary` })
  — overview of risk scores for all users in the account
```

### Without MCP Tools (Dashboard)

1. **View risk scores**: Zero Trust → Risk Score → Users
2. **Configure behaviors**: Zero Trust → Risk Score → Behaviors
   - Enable/disable specific behavior types
   - Adjust weights per behavior
3. **Create risk-based policies**:
   - Access → Applications → [App] → Policies → Add a rule with "Risk Score" condition
   - Gateway → HTTP Policies → Add a rule with user risk level selector

### Without MCP Tools (API)

```bash
# Get user risk score
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/zt_risk_scoring/{user_id}" \
  -H "Authorization: Bearer {api_token}" | jq '.result'

# List configured behaviors
curl -s "https://api.cloudflare.com/client/v4/accounts/{account_id}/zt_risk_scoring/behaviors" \
  -H "Authorization: Bearer {api_token}" | jq '.result'
```

## Validate

1. **Verify behavior configuration:**
   - With MCP: GET `/accounts/${accountId}/zt_risk_scoring/behaviors` — confirm expected behaviors are enabled with appropriate weights

2. **Check score accuracy:**
   - With MCP: GET `/accounts/${accountId}/zt_risk_scoring/${userId}` for known users — verify scores align with expected behavior
   - A user known to have triggered DLP violations should show elevated risk
   - A normal user with no anomalous behavior should show low risk

3. **Test policy integration:**
   - If using risk score in Access policies: verify that a high-risk user gets the expected policy action (step-up MFA, block, etc.)
   - If using risk score in Gateway policies: verify traffic routing changes for high-risk users

4. **Check for false positives:**
   - Users on VPNs or with legitimate multi-location travel may trigger impossible travel
   - Service accounts may trigger failed login behaviors during credential rotation
   - Review high-risk users to confirm the signals are genuine

## Operate

### Monitoring

- **High-risk user alerts**: Regularly review users in the High/Critical risk levels
- **Behavior weight tuning**: If a behavior generates too many false positives, reduce its weight rather than disabling it
- **Score trends**: A user's score trending upward over time is more meaningful than a one-time spike

### User Investigation Workflow

```
1. GET `/accounts/${accountId}/zt_risk_scoring/${userId}` → get score breakdown
2. Search CASB assets for user email across SaaS vendors
3. Check CASB findings related to this user
4. GET `/accounts/${accountId}/access/logs/access_requests` → review denied requests
5. Review Gateway logs for DLP violations
→ Consolidate into a risk assessment for the SOC team
```

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| All users show zero risk | No behaviors enabled, or signals not flowing | Check GET `/accounts/${accountId}/zt_risk_scoring/behaviors` — enable relevant behaviors. Verify Access/Gateway are logging events. |
| VPN users flagged as high risk | Impossible travel triggered by VPN IP changes | Reduce impossible travel weight, or add VPN exit IPs to an allow list |
| Service accounts flagged | Failed login behavior triggered during credential rotation | Consider excluding service accounts from risk scoring, or reduce failed login weight |
| Risk scores not updating | Behavior signals have latency — scores update asynchronously | Wait 1-4 hours after behavior events. Scores are not real-time. |

## Gotchas

1. **Risk scores are behavior-based, not finding-based.** A user with many CASB findings does NOT automatically have a high risk score. Risk scoring tracks behaviors (impossible travel, failed logins, DLP violations). CASB findings are a separate signal. Use both together for a complete picture.

2. **Scores update asynchronously.** After a behavior event (e.g., DLP violation), the risk score doesn't update instantly. Allow 1-4 hours for score recalculation.

3. **Impossible travel is the noisiest behavior.** Users on VPNs, corporate proxies, or traveling legitimately will trigger false positives. Tune the weight carefully and always investigate before acting.

4. **Risk score conditions in policies are level-based, not number-based.** You configure policies to match on Low/Medium/High/Critical levels, not on specific numeric thresholds. The level boundaries are system-defined.

5. **Disabling a behavior zeroes out its contribution.** If you disable impossible travel, all users' scores will drop by whatever that behavior was contributing. This can cause a sudden score decrease across many users.

6. **Risk scores require signal sources to be active.** DLP violation scoring requires DLP profiles and Gateway HTTP policies. Failed login scoring requires Access to be configured. If the upstream signal source isn't active, that behavior contributes nothing.

## Related Skills

- **casb-posture-triage**: Cross-reference CASB findings with risk scores for complete user investigation
- **access-app-setup**: Risk score can be used as an Access policy condition for step-up auth or blocking
- **gateway-policy-design**: Gateway policies can reference risk score levels for traffic routing decisions
- **dlp-profile-design**: DLP violations feed into risk scores — profiles must exist for this signal to work
