---
type: plan
title: "App uptime 403 flapping — CloudFront WAF IP-reputation false positives"
status: complete
owner: Engineering
last_reviewed: 2026-06-06
completed: 2026-06-06
applies_to: [triviaties]
controls: ["SOC2-CC6.6", "SOC2-CC7.1", "SOC2-CC7.2", "HIPAA-164.312(e)"]
---

# App uptime 403 flapping — CloudFront WAF IP-reputation false positives

**Affected:** `app.triviaties.com` (and all triviaties upptime checks)

## Symptom

Starting ~Jun 2 2026, the upptime monitor flapped: `Triviaties App` repeatedly
reported **down** then **back up** within ~30 min, multiple times a day
(issues #10–#21). Every "down" event was a fast **HTTP 403** (67–300 ms), never
a timeout or 5xx.

## Root cause

`app.triviaties.com` is served by **S3 + CloudFront** (distribution
`E2X1U96QKTJG7E`, alias `*.triviaties.com`). That distribution — along with ~10
other tenants (aiwcamp, abathhouse, naturalcare, bigquiz, …) — sits behind a
**shared WAFv2 Web ACL**:

```
CreatedByCloudFront-110a328f-236e-4e25-b96c-e96a72ba3169
scope=CLOUDFRONT  region=us-east-1  account=614746748396
```

Its rule at **priority 0**, `AWSManagedRulesAmazonIpReputationList`, was
**403-blocking the upptime checker**. GitHub-hosted runners draw from a large,
*rotating* pool of Azure IPs; some of those IPs are intermittently on Amazon's
threat-intel list. So some scheduled checks landed on a flagged IP → 403 (false
"down"), others on a clean IP → 200 (false "up") = flapping. The site was healthy
for real users the entire time.

### Proof (WAF logs → CloudWatch group `aws-waf-logs-test`)

WAF `BLOCK` events matched the down-issues to the second:

| Incident | WAF BLOCK (UTC) | Client IP (Azure) | Terminating rule |
|---|---|---|---|
| #21 | 15:53:58 (×3) | `52.154.20.50` | AmazonIpReputationList |
| #20 | 12:51:26 (×2) | `68.220.60.149` | AmazonIpReputationList |

Different Azure IP per incident = rotating runner pool, confirming the diagnosis.

## Fix

A top-priority **Allow** rule that lets only the monitor bypass the reputation
check, without weakening protection for anyone else.

1. **GitHub secret** `HEALTHCHECK_TOKEN` (high-entropy, set via `gh secret set`).
2. **`.upptimerc.yml`** — every triviaties check sends
   `X-Healthcheck-Token: $HEALTHCHECK_TOKEN`. Upptime substitutes the secret at
   runtime from `SECRETS_CONTEXT` in `.github/workflows/uptime.yml`.
3. **WAFv2** — new rule `Allow-Upptime-Healthcheck` at **priority 0** (managed
   rules renumbered to 1/2/3, none altered). It matches:
   - header `x-healthcheck-token` **EXACTLY** == the secret, **AND**
   - header `host` **ENDS_WITH** `triviaties.com` (LOWERCASE transform).

   The `host` clause limits blast radius: even a leaked token only bypasses WAF
   for triviaties hosts, never the other tenants on the shared ACL.

## Verification

- First runs on the new config (18:37:53, 18:38:28 UTC) from Azure IPs
  `52.161.60.228` / `145.132.102.49` carried the header and were **`ALLOW` via
  `Allow-Upptime-Healthcheck`** in the WAF logs.
- All four sites green at 200; Uptime CI run succeeded.

## Staging — investigated, deliberately NOT changed

`staging.triviaties.com` (distribution `E2ZF07TKGYHHDQ`, same shared WAF) is
monitored by **SentryUptimeBot**, not upptime. It does **not** share the problem:
Sentry checks from 3 **stable, clean GCP IPs** (`34.169.179.115`,
`34.85.249.57`, `35.237.134.233`) that don't hit the reputation list. 24h sample:
1685 `ALLOW`, **0** reputation blocks. A UA-based allow would be spoofable and,
because Allow terminates evaluation, would bypass *all* WAF rules — not worth it.
The existing host-scoped rule already covers staging, so **if** it ever flaps, the
only step is adding `X-Healthcheck-Token: <secret>` to the Sentry uptime monitor's
custom headers (Sentry UI) — no new WAF rule.

## Operational reference

- **Recheck health:** query `aws-waf-logs-test` for
  `terminatingRuleId="Allow-Upptime-Healthcheck"` (expect `ALLOW`), or for
  `BLOCK` on `AWSManagedRulesAmazonIpReputationList`.
- **Rotate the token:** update the `HEALTHCHECK_TOKEN` GitHub secret and the
  `SearchString` (base64) in the WAF rule together.
- **Distributions:** prod `E2X1U96QKTJG7E` (`*.triviaties.com`),
  staging `E2ZF07TKGYHHDQ`.
- **Related (tag-compliance):** `plans/aws-waf-coverage.md` (Phase 3 IP-reputation
  caveat references this incident) and `status-page/README.md` ("Known limitations"
  WAF-flapping note). The superseded `plans/status-page-strategy.md` points to that
  README as the live source of truth.
